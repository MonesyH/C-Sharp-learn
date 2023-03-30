# 一、InlineData
```
public class MathUtilsTests
{
    [Theory]
    [InlineData(2, 3, 5)]
    [InlineData(-2, 3, 1)]
    [InlineData(0, 0, 0)]
    public void Add_ShouldReturnCorrectSum(int a, int b, int expectedSum)
    {
        // Arrange
        MathUtils mathUtils = new MathUtils();

        // Act
        int sum = mathUtils.Add(a, b);

        // Assert
        Assert.Equal(expectedSum, sum);
    }
}

```
# 二、ClassData
```
public class MathUtilsTests
{
    public static IEnumerable<object[]> TestData()
    {
        yield return new object[] { 2, 3, 5 };
        yield return new object[] { -2, 3, 1 };
        yield return new object[] { 0, 0, 0 };
    }

    [Theory]
    [ClassData(typeof(MathUtilsTestData))]
    public void Add_ShouldReturnCorrectSum(int a, int b, int expectedSum)
    {
        // Arrange
        MathUtils mathUtils = new MathUtils();

        // Act
        int sum = mathUtils.Add(a, b);

        // Assert
        Assert.Equal(expectedSum, sum);
    }
}

```
