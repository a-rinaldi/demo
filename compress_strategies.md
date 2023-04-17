The **strategy pattern** is used in the provided code (Java + Spring) to provide a flexible and extensible way of implementing different compression strategies for different file formats. This is because different file formats may require different algorithms and techniques to compress them effectively. By using the strategy pattern, the code can easily support the addition of new compression strategies without the need to modify the existing code, thereby ensuring that the code remains maintainable and extensible.

```java
@Bean
public CompressStrategies compressStrategies() {
        final Map<Format, CompressStrategy> strategies = Maps.<Format, CompressStrategy>builder()
                .add(Format.JPG, new JpgCompressStrategy())
                .add(Format.JPEG, new JpgCompressStrategy())
                .add(Format.GIF, new GifCompressStrategy())
                .add(Format.PNG, new PngCompressStrategy())
                .add(Format.NO_FORMAT, new NoCompressStrategy())
                .toMap();
        return new CompressStrategies(strategies);
    }
```
This bean enables the selection of a suitable compression strategy based on the format of the uploaded file. The CompressStrategies class is responsible for mapping each format to its respective compression strategy.
```java
public class CompressStrategies {

    private final Map<Format, CompressStrategy> strategies;

    public CompressStrategies(Map<Format, CompressStrategy> strategies) {
        this.strategies = strategies;
    }

    public CompressStrategy get(Format format) {
        return Optional.ofNullable(strategies.get(format))
                .orElseThrow(() -> new IllegalArgumentException(String.format("No strategies available for format %s", format)));
    }
}
```
The CompressStrategy interface defines the contract that each compression strategy must follow.
```java
public interface CompressStrategy {

    public byte[] compress(byte[] bytes);

}
```
An implementation of the CompressStrategy interface is shown below for the PNG format.
```java
public class PngCompressStrategy implements CompressStrategy {

    @Override
    public byte[] compress(byte[] bytes) {
        try (Reader reader = new InputStreamReader(new ByteArrayInputStream(bytes))) {
                        [...]
       }
      }
   }
}
```
Click to watch the final output [![video](https://img.youtube.com/vi/dfZ3kVSDRg4/maxresdefault.jpg)](https://www.youtube.com/watch?v=dfZ3kVSDRg4)


