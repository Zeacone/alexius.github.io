---
title: Swift字符串的长度问题
date: 2018-05-14 14:16:03
tags: [iOS, Swift, String, NSString]
---

### 获取的字符串长度不一样？

在 Swift 中获取字符串的长度很简单，只需要调用 `count` 属性即可。

```swift
let string = "这是🐘，它的👃很长"
print("string 的长度为： \(string.count)")
// 输出: "string 的长度为： 9"
```

这在日常的使用中是没有什么问题的，直到我在项目中遇到了一个评论 emoji 乱码的问题。  
问题表现为，在评论的末尾如果是 emoji 的话， 则会变成无法显示的❓，如果 emoji 出现在其它地方，则很少出现这个显示不了的问题。
后来经过多次的一步步调试，终于确定是在将字符串转为富文本的时候出现的问题。

因为评论中包含链接，用户名，以及要调节行高和字间距等等，所以必须要使用富文本来显示，在此处，就简单地写下大概的代码：
```swift
func convertToAttributedString(with string: String) {
    let attrString = NSMutableAttributedString(string: string)
    let range = NSMakeRange(0, string.count) // 1
    attrString.addAttribute(..., value: ..., range: range)
    ...
    return attrString
}

convertToAttributedString(with: "今天天气真好啊😄")
```

问题就出现在注释1的地方，先来看看 Swift 的官方文档。

> Extended grapheme clusters can be composed of multiple Unicode scalars. This means that different characters—and different representations of the same character—can require different amounts of memory to store. Because of this, characters in Swift don’t each take up the same amount of memory within a string’s representation. As a result, the number of characters in a string can’t be calculated without iterating through the string to determine its extended grapheme cluster boundaries. If you are working with particularly long string values, be aware that the count property must iterate over the Unicode scalars in the entire string in order to determine the characters for that string.
>
> The count of the characters returned by the count property isn’t always the same as the length property of an NSString that contains the same characters. The length of an NSString is based on the number of 16-bit code units within the string’s UTF-16 representation and not the number of Unicode extended grapheme clusters within the string.

意思就是有个东西叫做 `Extended Grapheme Clusters`， 而这个 `Extended Grapheme Clusters` 是可以由多个 Unicode 标量组成。因此不同的字符或者同一字符的不同表现形式可能需要不同数量的内存来存储。这样就造成了 `String` 的 `count` 属性无法准确表示实际所占用的内存大小，而只是我们肉眼所能看到的字符长度。而 `NSString` 使用的是 UTF-16 字符编码方式，`length` 表示的是16位代码单元的数量而不是扩展字符集的数量。

现在再回头看我所遇到的问题就比较清晰明了了，`NSMutableAttributedString` 与 `NSString` 的表现形式一样使用 UTF-16。
上面的例子中 `string.count` 的值为 8，而 `attrString` 在添加属性的时候，按照这个 `range`，并不能完整地将 emoji 转为富文本，因此 emoji 才会无法显示。

那么如何修改呢，其实非常简单，只要在计算 range 的时候将 String 转为 NSString，或者使用 UTF-16 的数量。

```swift
func convertToAttributedString(with string: String) {
    let attrString = NSMutableAttributedString(string: string)
    let range1 = NSMakeRange(0, NSString(string: string).length)
    let range2 = NSMakeRange(0, string.utf16.count)
    attrString.addAttribute(..., value: ..., range: range1)
    ...
    return attrString
}

convertToAttributedString(with: "今天天气真好啊😄")
```
现在使用 `range1` 或者 `range2` 转换为富文本就不会再出现末尾 emoji 乱码的问题了。



### 中英文限制字数的问题
既然都说到字符串长度的问题了，顺便再讨论下另外以一个问题。  

相信在日常的工作中，我们都不免会遇到要限制输入字数的问题。通用的做法都是在 `textField(_ textField: UITextField, shouldChangeCharactersIn range: NSRange, replacementString string: String) -> Bool` 中计算 textField.text.count 和 string.count 之和，如果数字超过了限制的最大字数，则返回 false，不再允许用户输入。

这种做法一般来说没什么问题，但是如果老板告诉你中文限制100字，英文限制200字，那么应该怎么办呢？

我们先来看看，不同的编码方式是怎么来存储字符的。
- UTF-8：一个英文字符占用一个字节，一个中文占用三个字节。
- UTF-16：一个英文字母字符或一个汉字字符存储都需要 2 个字节（Unicode 扩展区的一些汉字存储需要4 个字节）
- UTF-32：使用 4 个字节表示一个字符。
- GBK：一个英文占用一个字节，一个中文占用两个字节。
- GB2312：一个英文占用一个字节，一个中文占用两个字节。

***此处只是简单地认识一下不同编码方式下中英文所占用的内存空间大小，想更深入了解请查询字符编码相关详细资料。***

从上面可以看出，如果使用GBK 编码和 GB2312 编码，那么只需要计算出所占用的字节数， 就可以变相地达到限制中英文不同字数的需求。比如说，如果在 `UITextField` 的代理中限制字节数为 20，那么就只能输入 10 个中文文字或 20 个英文字母或者 5 个中文和 10 个英文字母。

下面则是计算所占用字节数的计算方法：
```swift
func getBytesCount(of string: String) -> Int {
    let encoding = CFStringConvertEncodingToNSStringEncoding(CFStringEncoding(CFStringEncodings.GB_18030_2000.rawValue))
    let data = string.data(using: String.Encoding(rawValue: encoding))!
    return data.count
}
```


---

参考资料：

[Swift 文档](https://docs.swift.org/swift-book/LanguageGuide/StringsAndCharacters.html)