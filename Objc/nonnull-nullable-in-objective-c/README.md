# Objective-C 中 nullable、__nullable、_Nullable 的区别

来源：https://kangzubin.com/nullability-and-objective-c/

## 缘由

在 Swift 中，我们会使用 `?` 和 `!` 去显式声明一个对象或者方法的参数是 `optional` 还是 `non-optional`，而在 Objective-C 中则没有这一区分，这样就会带来一个问题：在 Swift 与Objective-C 混编时，Swift 编译器并不知道一个 Objective-C 对象或者一个方法的参数到底是 `optional` 还是 `non-optional`，因此这种情况下编译器会隐式地都当成是 `non-optional` 来处理，这显然是不太好的。



## 挖坑

为了解决这个问题，苹果在 Xcode 6.3 引入了一个 Objective-C 的新特性：**Nullability Annotations**，这一新特性的核心是两个新的类型修饰：`__nullable` 和 `__nonnull`。从字面上我们可知，`__nullable` 表示对象可以是 NULL 或 nil，而 `__nonnull` 表示对象不应该为空。当我们不遵循这一规则时，编译器就会给出警告。在 Xcode 7 中，为了避免与第三方库潜在的冲突，苹果把 `__nonnull/__nullable` 改成 `_Nonnull/_Nullable`。再加上苹果同样支持了没有下划线的写法 `nonnull/nullable`，于是就造成现在有三种写法这样混乱的局面。但是这三种写法本质上都是互通的，只是放的位置不同，举例如下：

方法返回值修饰：

```
- (nullable NSString *)method;
- (NSString * __nullable)method;
- (NSString * _Nullable)method;
```

声明属性的修饰：

```
@property (nonatomic, copy, nullable) NSString *aString;
@property (nonatomic, copy) NSString * __nullable aString;
@property (nonatomic, copy) NSString * _Nullable aString;
```

方法参数修饰：

```
- (void)methodWithString:(nullable NSString *)aString;
- (void)methodWithString:(NSString * _Nullable)aString;
- (void)methodWithString:(NSString * __nullable)aString;
```

而对于 `双指针类型对象` 、`Block 的返回值`、`Block 的参数` 等，这时候就不能用 `nonnull/nullable` 修饰，只能用带下划线的 `__nonnull/__nullable` 或者 `_Nonnull/_Nullable`：

```
- (void)methodWithError:(NSError * _Nullable * _Nullable)error
- (void)methodWithError:(NSError * __nullable * __null_unspecified)error;
// 以及其他的组合方式
- (void)methodWithBlock:(nullable void (^)())block; 
// 注意上面的 nullable 用于修饰方法传入的参数 Block 可以为空，而不是修饰 Block 返回值；
- (void)methodWithBlock:(void (^ _Nullable)())block;
- (void)methodWithBlock:(void (^ __nullable)())block;
- (void)methodWithBlock:(nullable id __nonnull (^)(id __nullable params))block;
// 注意上面的 nullable 用于修饰方法传入的参数 Block 可以为空，而 __nonnull 用于修饰 Block 返回值 id 不能为空；
- (void)methodWithBlock:(id __nonnull (^ __nullable)(id __nullable params))block;
- (void)methodWithBlock:(id _Nonnull (^ _Nullable)(id _Nullable params))block;
// the method accepts a nullable block that returns a nonnull value
// there are some more combinations here, you get the idea
```

以上基本上罗列了绝大部分的使用场景，但看完我们还是一脸懵逼啊，仍然不清楚什么时候应该用哪个修饰符！

在看了原生 iOS SDK 里 Foundation 和 UIKit 的头文件以及苹果的博文[《Nullability and Objective-C》](https://developer.apple.com/swift/blog/?id=25)，我们总结如下使用规范：

- **对于属性、方法返回值、方法参数的修饰，使用：**`nonnull/nullable`；
- **对于 C 函数的参数、Block 的参数、Block 返回值的修饰，使用：**`_Nonnull/_Nullable`，**建议弃用** ~~`__nonnull/__nullable`~~。

## Nonnull Audited Regions

如果需要每个属性或每个方法都去指定 `nonnull` 和 `nullable`，将是一件非常繁琐的事。苹果为了减轻我们的工作量，专门提供了两个宏：`NS_ASSUME_NONNULL_BEGIN` 和 `NS_ASSUME_NONNULL_END`。在这两个宏之间的代码，所有简单指针对象都被假定为 `nonnull`，因此我们只需要去指定那些 `nullable`指针对象即可。如下代码所示：

```
NS_ASSUME_NONNULL_BEGIN

@interface myClass ()

@property (nonatomic, copy) NSString *aString;

- (id)methodWithString:(nullable NSString *)str;

@end

NS_ASSUME_NONNULL_END
```

在上面的代码中，`aString` 属性默认是 `nonnull` 的，`methodWithString:` 方法的返回值也是 `nonnull`，而方法的参数 `str` 被显式指定为 `nullable`。

不过，为了安全起见，苹果还制定了以下几条规则：

- 通过 `typedef` 定义的类型的 `nullability` 特性通常依赖于上下文，即使是在 Audited Regions 中，也不能假定它为 `nonnull`；
- 对于复杂的指针类型（如 `id *`）必须显式去指定是 `nonnull` 还是 `nullable`。例如，指定一个指向 `nullable` 对象的 `nonnull` 指针，可以使用 `__nullable id * __nonnull`；
- 我们经常使用的 `NSError **` 通常是被假定为一个指向 `nullable` NSError 对象的`nullable` 指针。

## 疑问

虽然在 Xcode 7 里面，苹果建议我们放弃使用 `__nonnull/__nullable`，改用 `_Nonnull/_Nullable` 来修饰对象可否为空，但即使是在最新 iOS 9.3 SDK 的 Foundation 和 UIKit 的头文件里我们可以看到官方原生类的方法参数仍然在用 `__nonnull/__nullable` 修饰。另外为什么已经有了 `nonnull/nullable`，为什么还要增加 `_Nonnull/_Nullable` ？这到底是出于什么考虑？苹果在它的博文 [《Nullability and Objective-C》](https://developer.apple.com/swift/blog/?id=25) 中也没有具体解释，于是 StackOverflow 上有个关于此问题的讨论：[Difference between nullable, __nullable and _Nullable in Objective-C](http://stackoverflow.com/questions/32452889/difference-between-nullable-nullable-and-nullable-in-objective-c)。

## References

[Nullability and Objective-C](https://developer.apple.com/swift/blog/?id=25)
[会报编译器警告的 Xcode 6.3 新特性：Nullability Annotations](http://www.cocoachina.com/ios/20150603/11989.html)
[Objective-C 关键字 _Nullable 和 _Nonnull](http://blog.csdn.net/top_roboo/article/details/50250785)
[Objective-C 的 Nullability 特性](http://xuexuefeng.com/nullability-feature-of-objective-c/)