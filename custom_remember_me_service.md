This code defines a custom Remember Me service that extends the **TokenBasedRememberMeServices** class provided by Spring Security.
The CustomRememberMeService overrides the **onLoginSuccess** method to add a Remember Me cookie to the user's browser when they successfully log in. The cookie contains the user's username, a mode (enumeration), the expiry time of the token, and a signature.
The mode defines if the user is an individual or a company
```java
public static enum Mode {
        COMPANY,
        USER
    }
```

The **makeTokenSignature** method calculates the signature by concatenating the username, mode, token expiry time, password, and a key (passed to the constructor) with a colon (":") separator, then hashing the resulting string using MD5. The resulting hash is converted to a string of hexadecimal digits and returned as the signature.

The **processAutoLoginCookie** method overrides the base implementation to extract the Remember Me cookie from the request and validate it. If the cookie is valid, the method uses the UserDetailsFromCustomer class to load the user's details (e.g., email, password) from the backend, checks that the token has not expired, and validates the signature in the cookie using the makeTokenSignature method. Finally, the method returns the UserDetails object containing the user's details.

```java
package net.optionfactory.peekaboox.web;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.Arrays;
import java.util.Date;
import java.util.Optional;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import net.optionfactory.peekaboox.core.customers.Customer;
import net.optionfactory.peekaboox.core.customers.Customer.Mode;
import net.optionfactory.peekaboox.core.customers.CustomerResponse;
import net.optionfactory.peekaboox.core.customers.CustomersFacade;
import net.optionfactory.peekaboox.core.marketplace.MarketplaceCompanies;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.security.crypto.codec.Hex;
import org.springframework.security.crypto.codec.Utf8;
import static org.springframework.security.web.authentication.rememberme.AbstractRememberMeServices.TWO_WEEKS_S;
import org.springframework.security.web.authentication.rememberme.InvalidCookieException;
import org.springframework.security.web.authentication.rememberme.RememberMeAuthenticationException;
import org.springframework.security.web.authentication.rememberme.TokenBasedRememberMeServices;
import org.springframework.util.StringUtils;

public class CustomRememberMeService extends TokenBasedRememberMeServices {

    public CustomRememberMeService(String key, UserDetailsService userDetailsService) {
        super(key, userDetailsService);
    }

    @Override
    public void onLoginSuccess(HttpServletRequest request, HttpServletResponse response,
            Authentication successfulAuthentication) {

        String username = retrieveUserName(successfulAuthentication);
        String password = retrievePassword(successfulAuthentication);
        final Mode mode = Mode.valueOf(request.getParameter("mode"));
        if (!StringUtils.hasLength(username)) {
            logger.debug("Unable to retrieve username");
            return;
        }

        if (!StringUtils.hasLength(password)) {
            UserDetails user = getUserDetailsService().loadUserByUsername(username);
            password = user.getPassword();

            if (!StringUtils.hasLength(password)) {
                logger.debug("Unable to obtain password for user: " + username);
                return;
            }
        }

        int tokenLifetime = calculateLoginLifetime(request, successfulAuthentication);
        long expiryTime = System.currentTimeMillis();
        // SEC-949
        expiryTime += 1000L * (tokenLifetime < 0 ? TWO_WEEKS_S : tokenLifetime);

        String signatureValue = makeTokenSignature(expiryTime, username, password, mode);

        setCookie(new String[]{username, mode.name(), Long.toString(expiryTime), signatureValue},
                tokenLifetime, request, response);

        if (logger.isDebugEnabled()) {
            logger.debug("Added remember-me cookie for user '" + username
                    + "', expiry: '" + new Date(expiryTime) + "'");
        }
    }

    /**
     * Calculates the digital signature to be put in the cookie. Value is MD5
     * ("username:mode:tokenExpiryTime:password:key")
     */
    protected String makeTokenSignature(long tokenExpiryTime, String username,
            String password, Mode mode) {
        String data = String.format("%s:%s:%s:%s:%s", username, mode, tokenExpiryTime, password, getKey());
        MessageDigest digest;
        try {
            digest = MessageDigest.getInstance("MD5");
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("No MD5 algorithm available!");
        }
        return new String(Hex.encode(digest.digest(data.getBytes())));
    }

    @Override
    protected UserDetails processAutoLoginCookie(String[] cookieTokens, HttpServletRequest request, HttpServletResponse response) throws RememberMeAuthenticationException, UsernameNotFoundException {
        if (cookieTokens.length != 4) {
            throw new InvalidCookieException("Cookie token did not contain 4"
                    + " tokens, but contained '" + Arrays.asList(cookieTokens) + "'");
        }
        Mode mode = Mode.valueOf(cookieTokens[1]);
        ((UserDetailsFromCustomer) this.getUserDetailsService()).setMode(mode);
        long tokenExpiryTime;
        try {
            tokenExpiryTime = new Long(cookieTokens[2]).longValue();
        } catch (NumberFormatException nfe) {
            throw new InvalidCookieException(
                    "Cookie token[1] did not contain a valid number (contained '"
                    + cookieTokens[2] + "')");
        }
        if (isTokenExpired(tokenExpiryTime)) {
            throw new InvalidCookieException("Cookie token[2] has expired (expired on '"
                    + new Date(tokenExpiryTime) + "'; current time is '" + new Date()
                    + "')");
        }

        UserDetails userDetails = getUserDetailsService().loadUserByUsername(
                cookieTokens[0]);

        String expectedTokenSignature = makeTokenSignature(tokenExpiryTime,
                userDetails.getUsername(), userDetails.getPassword(), mode);

        if (!equals(expectedTokenSignature, cookieTokens[3])) {
            throw new InvalidCookieException("Cookie token[3] contained signature '"
                    + cookieTokens[3] + "' but expected '" + expectedTokenSignature + "'");
        }

        return userDetails;
    }

    public static class UserDetailsFromCustomer implements UserDetailsService {

        private CustomersFacade customersFacade;
        private Mode mode;

        public UserDetailsFromCustomer(CustomersFacade customersFacade) {
            this.customersFacade = customersFacade;
        }

        public Mode getMode() {
            return mode;
        }

        public void setMode(Mode mode) {
            this.mode = mode;
        }

        @Override
        public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
            return customersFacade.lookup(MarketplaceCompanies.HELP4U, email, this.getMode())
                    .orElseThrow(() -> new UsernameNotFoundException("User not found"));
        }
    }

    /**
     * Constant time comparison to prevent against timing attacks.
     */
    private static boolean equals(String expected, String actual) {
        byte[] expectedBytes = bytesUtf8(expected);
        byte[] actualBytes = bytesUtf8(actual);
        if (expectedBytes.length != actualBytes.length) {
            return false;
        }

        int result = 0;
        for (int i = 0; i < expectedBytes.length; i++) {
            result |= expectedBytes[i] ^ actualBytes[i];
        }
        return result == 0;
    }

    private static byte[] bytesUtf8(String s) {
        if (s == null) {
            return null;
        }
        return Utf8.encode(s);
    }

}
```
CustomSecurityConfig extends WebSecurityConfigurerAdapter from Spring Security and overrides the **configure** method to configure security. Specifically, it configures the HttpSecurity object to enable the Remember Me functionality using a custom implementation of the RememberMeServices interface called CustomRememberMeService. This service requires a secret key which is specified using the key method.

```java
@Configuration
public static class CustomSecurityConfig extends WebSecurityConfigurerAdapter {
	@Override
	protected void configure(HttpSecurity http) throws Exception {
			final AuthenticationProvider usersAuthenticationProvider = new CustomAuthenticationProvider(passwordEncoder, t -> this.lookupUserDetails(t));
			http
							[...] //matcher and handles
							.and()
							.rememberMe()
							.rememberMeServices(new CustomRememberMeService(rememberMeSecret, new UserDetailsFromCustomer(customerFacade)))
							.key(rememberMeSecret)
							.and()
	}

}
```

Final output [video](https://www.youtube.com/watch?v=Vx90mELs0SM)
