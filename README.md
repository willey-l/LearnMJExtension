## 最简单的字典

首先,从最简单的字典开始.

``` objective-c
NSDictionary *dict = @{
                           @"name" : @"Jack",
                           @"icon" : @"lufy.png",
                           @"age" : @"20",
                           @"height" : @1.55,
                           @"money" : @"100.9",
                           @"sex" : @(SexFemale),
                           @"gay" : @"1"
						}
```

目标是拿到字典里的值对`User`模型进行赋值.模型的`属性`对应字典的`键`.

``` objective-c
typedef enum {
    SexMale,
    SexFemale
} Sex;

@interface User : NSObject

/** 名称 */
@property (copy, nonatomic) NSString *name;
/** 头像 */
@property (copy, nonatomic) NSString *icon;
/** 年龄 */
@property (assign, nonatomic) unsigned int age;
/** 身高 */
@property (copy, nonatomic) NSString *height;
/** 财富 */
@property (strong, nonatomic) NSNumber *money;
/** 性别 */
@property (assign, nonatomic) Sex sex;
/** 同性恋 */
@property (assign, nonatomic, getter=isGay) BOOL gay;

@end
```

最直接的方法是:

``` objective-c
 User *user = [[User alloc] init];
 user.name = dict[@"name"];
 user.icon = dict[@"icon"];
 ....
```

假如属性数量一多,人工手写大量`样板代码`将耗费大量时间和精力,毫无意义.

如果要写一个框架自动帮我们转模型出来,大致思路如下:

**1.遍历模型中的`属性`,然后拿到`属性名`作为`键值`去字典中寻找`值`.**

**2.找到`值`后根据模型的`属性`的`类型`将`值`转成正确的`类型`**

**3.赋值**

------

首先进行**第一步**:

> 遍历模型中的`属性`,然后拿到`属性名`作为`键值`去字典中寻找`值`.

伪代码:

``` objective-c
[模型类 遍历属性的方法];
```

为了方便使用,创建一个叫`NSObject+Property`的分类.写一个获取所有属性的方法.

``` objective-c
@interface NSObject (Property)
+ (NSArray *)properties;
@end
```

假设我们看不见一个类的`.h`和`.m`,有什么办法可以获取它所有属性的名字和类型呢?答案是通过`运行时机制`.当在实现`+ (NSArray *)properties`方法时,需要**导入运行时库**.然后使用库中的API提供的函数得到一个类的方法列表.

``` objective-c
objc_property_t *class_copyPropertyList(Class cls, unsigned int *outCount)
```

返回的是叫`objc_property_t`的一个结构体指针,并且通过传入值引用能够得到属性的个数.

``` objective-c
#import "NSObject+Property.h"
#import <objc/runtime.h>

@implementation NSObject (Property)
+ (NSArray *)properties{
    NSArray *propertiesArray = [NSMutableArray array];
  	// 1.获得所有的属性
    unsigned int outCount = 0;
    objc_property_t *properties = class_copyPropertyList(self, &outCount);
  	// .....
    return propertiesArray;
}
@end
```

来到这里已经获取到了属性列表,那么`objc_property_t`指向的结构体内部是怎样的呢.通过搜寻`<objc/runtime.h>`头文件并看不到`objc_property_t`的定义的.但好在`runtime`开源,我们搜寻到了相关的定义.

``` objective-c
typedef struct property_t *objc_property_t;

struct property_t {
    const char *name;
    const char *attributes;
};
```

由于知道了结构体的内部构造,就可以获取内部的成员变量.例如一下方法:

``` objective-c
typedef struct property_t {
    const char *name;
    const char *attributes;
} *propertyStruct;

@implementation NSObject (Property)

+ (NSArray *)properties{
    NSArray *propertiesArray = [NSMutableArray array];
    // 1.获得所有的属性
    unsigned int outCount = 0;
    objc_property_t *properties = class_copyPropertyList(self, &outCount);

    for (int i = 0; i < outCount; i++) {
        objc_property_t property = properties[i];
        NSLog(@"name:%s---attributes:%s",((propertyStruct)property)->name,((propertyStruct)property)->attributes);
    }

    return propertiesArray;
}
@end
```

在外部调用`+ (NSArray *)properties`方法能够打印出一个类的全部属性,如:

``` objective-c
NSArray *propertyArray = [User properties];
```

得到控制台输出:


![](http://upload-images.jianshu.io/upload_images/651640-240c90352885fb9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从输出中可以看到该结构体的`name`成员表示成员属性的名字,`attributes`表示成员属性中的一些信息(如是什么类,原子性还是非原子性,是strong还是weak还是copy,生成的成员变量名等信息)...

从苹果的[官方文档(Objective-C Runtime Programming Guide)](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtPropertyIntrospection.html)可以得知,`attributes`是一个类型编码字符串.可以使用`property_getAttributes `函数获得这个类型编码字符串.这个字符串以`T`作为开始,接上`@encode`类型编码和一个逗号,以`V`接上实例变量名作为结尾,在它们之间是一些其他信息,以逗号分割.具体内容可以看官方文档中详细的表格.



在实际赋值过程中,我们并不用关心该属性的内存管理语义,生成的成员变量名,或者其他什么信息.在`attributes`中,只需要知道它所属的`类`或者`类型`,即`T`至`第一个逗号之前`中间的内容,如果是类的话还需要将`@"`和`"`去掉.

实际上,框架提供的运行时库已经给我们提供获取`成员属性名字`和`成员属性的属性们`的函数了.

``` objective-c
NSLog(@"name:%s---attributes:%s",property_getName(property),
      							 property_getAttributes(property));
```

从源码中可以看到这两个函数的内部是这样写的:

``` objective-c
const char *property_getName(objc_property_t prop)
{
    return prop->name;
}

const char *property_getAttributes(objc_property_t prop)
{
    return prop->attributes;
}
```



**再回顾前面说的思路,这时会更清晰:**

**1.拿到模型的属性名(注意属性名和成员变量名的区别),和对应的类或类型**.

**2.用该属性名作为键去字典中寻找对应的值.**

**3.拿到值后将值转换为属性对应的类或类型.**

**4.赋值.**

现在已经进行到第一步,并且拿到了属性名,但是类或类型还要进一步截取,截取方法如下:

``` objective-c
for (int i = 0; i < outCount; i++) {
        objc_property_t property = properties[i];
  		
  		// 为了以后方便,将C字符串转换成OC对象
        NSString *name = @(property_getName(property));
        NSString *attributes = @(property_getAttributes(property));

        NSUInteger loc = 1;
        NSUInteger len = [attributes rangeOfString:@","].location - loc;
        NSString *type = [attributes substringWithRange:NSMakeRange(loc, len)];
        NSLog(@"%@",type);
    }
```

控制台结果显示我们能够截取到其中的类型了.


![](http://upload-images.jianshu.io/upload_images/651640-983822a55237d82b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



回归我们拿到这些类型的初衷,是为了是用字典中的`值`类型与模型中要进行赋值的`属性`的类型进行对比,想要对比,需要拿到属性的`class`,因此需要将这些编码转换成一个**表示类型的类**,创建一个类用来包装类型.

``` objective-c
/**
 *  包装一种类型
 */
@interface MJPropertyType : NSObject

/** 是否为id类型 */
@property (nonatomic, readonly, getter=isIdType) BOOL idType;

/** 是否为基本数字类型：int、float等 */
@property (nonatomic, readonly, getter=isNumberType) BOOL numberType;

/** 是否为BOOL类型 */
@property (nonatomic, readonly, getter=isBoolType) BOOL boolType;

/** 对象类型（如果是基本数据类型，此值为nil） */
@property (nonatomic, readonly) Class typeClass;

@end
```



OC对象可以通过`Class`来表示类型,而基本数据类型只能用布尔来标识.

把这些名字和类型遍历出来,肯定是为了以后有用,所以需要把它们存起来,由于它们是一个”整体",所以还是设计一个类将他们包装起来比较好.创建一个包装成员属性的类—`MJProperty`.

``` objective-c
@interface MJProperty : NSObject

/** 成员属性的名字 */
@property (nonatomic, readonly) NSString *name;
/** 成员属性的类型 */
@property (nonatomic, readonly) MJPropertyType *type;

@end
```

这时,代码就可以进行重构了,将属于不同类的功能封装到对应的类上,让`MJProperty`提供一个类方法用于返回一个将`objc_property_t`进行包装的类.

``` objective-c
 for (int i = 0; i < outCount; i++) {
        objc_property_t property = properties[i];

        MJProperty *propertyObj = [MJProperty propertyWithProperty:property];
 }
```

`propertyWithProperty:`方法的实现如下:

``` objective-c
+ (instancetype)propertyWithProperty:(objc_property_t)property{
    return  [[MJProperty alloc] initWithProperty:property];
}


- (instancetype)initWithProperty:(objc_property_t)property{
    if (self = [super init]) {
        _name = @(property_getName(property));
        _type = [MJPropertyType propertyTypeWithAttributeString:@(property_getAttributes(property))];;
    }
    return self;
}
```

`MJPropertyType`也提供类方法用于包装类型:

``` objective-c
+ (instancetype)propertyTypeWithAttributeString:(NSString *)string{
    return [[MJPropertyType alloc] initWithTypeString:string];
}

- (instancetype)initWithTypeString:(NSString *)string
{
    if (self = [super init])
    {
        NSUInteger loc = 1;
        NSUInteger len = [string rangeOfString:@","].location - loc;
        NSString *type = [string substringWithRange:NSMakeRange(loc, len)];

        NSLog(@"%@",type);
    }
    return self;
}
```

重构完成之后,结构显得更加清晰.更有利于接下来的工作.下面继续完成`type`的提取.

------

![](http://upload-images.jianshu.io/upload_images/651640-240c90352885fb9c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面获取到的这些类型,是类型编码,在苹果文档中告诉了我们编码对应的类型:



![](http://upload-images.jianshu.io/upload_images/651640-449ad8ff10affa5f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据这个对应关系的图表,我们将常用的几个编码定义成常量字符串或者宏表示它所对应的类型,便于编码和阅读:

``` objective-c
/**
 *  成员变量类型（属性类型）
 */
NSString *const MJPropertyTypeInt = @"i";
NSString *const MJPropertyTypeShort = @"s";
NSString *const MJPropertyTypeFloat = @"f";
NSString *const MJPropertyTypeDouble = @"d";
NSString *const MJPropertyTypeLong = @"q";
NSString *const MJPropertyTypeChar = @"c";
NSString *const MJPropertyTypeBOOL1 = @"c";
NSString *const MJPropertyTypeBOOL2 = @"b";
NSString *const MJPropertyTypePointer = @"*";

NSString *const MJPropertyTypeIvar = @"^{objc_ivar=}";
NSString *const MJPropertyTypeMethod = @"^{objc_method=}";
NSString *const MJPropertyTypeBlock = @"@?";
NSString *const MJPropertyTypeClass = @"#";
NSString *const MJPropertyTypeSEL = @":";
NSString *const MJPropertyTypeId = @"@";
```

设置完后,就可以进行提取类型了.

``` objective-c
- (instancetype)initWithTypeString:(NSString *)string
{
    if (self = [super init])
    {
        NSUInteger loc = 1;
        NSUInteger len = [string rangeOfString:@","].location - loc;
        NSString *typeCode = [string substringWithRange:NSMakeRange(loc, len)];
        [self getTypeCode:typeCode];
        NSLog(@"%@",typeCode);
    }
    return self;
}

- (void)getTypeCode:(NSString *)code
{
    if ([code isEqualToString:MJPropertyTypeId]) {
        _idType = YES;
    } else if (code.length > 3 && [code hasPrefix:@"@\""]) {
        // 去掉@"和"，截取中间的类型名称
        _code = [code substringWithRange:NSMakeRange(2, code.length - 3)];
        _typeClass = NSClassFromString(_code);
        _numberType = (_typeClass == [NSNumber class] || [_typeClass isSubclassOfClass:[NSNumber class]]);
    }

    // 是否为数字类型
    NSString *lowerCode = _code.lowercaseString;
    NSArray *numberTypes = @[MJPropertyTypeInt, MJPropertyTypeShort, MJPropertyTypeBOOL1, MJPropertyTypeBOOL2, MJPropertyTypeFloat, MJPropertyTypeDouble, MJPropertyTypeLong, MJPropertyTypeChar];
    if ([numberTypes containsObject:lowerCode]) {
        _numberType = YES;

        if ([lowerCode isEqualToString:MJPropertyTypeBOOL1]
            || [lowerCode isEqualToString:MJPropertyTypeBOOL2]) {
            _boolType = YES;
        }
    }
}
```

至此,一个`MJProperty`的骨架就大致搭好了.


![13F73F26-1195-43BC-BC98-FF2641B7DA58.png](http://upload-images.jianshu.io/upload_images/651640-b1dba063f80e8336.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当想要使用字典转模型的功能时,提供一个类方法方便转换,该方法放在`NSObject+keyValue2object`分类中,该分类负责字典转模型的方法实现.

``` objective-c
@implementation NSObject (keyValue2object)

+ (instancetype)objectWithKeyValues:(id)keyValues{
    if (!keyValues) return nil;
    return [[[self alloc] init] setKeyValues:keyValues];
}

- (instancetype)setKeyValues:(id)keyValues{
    NSArray *propertiesArray = [self.class properties];
    for (MJProperty *property in propertiesArray) {
        MJPropertyType *type = property.type;
        Class typeClass = type.typeClass;
        if (type.isBoolType) {
            NSLog(@"bool");
        }else if (type.isIdType){
            NSLog(@"ID");
        }else if (type.isNumberType){
            NSLog(@"Number");
        }else{
            NSLog(@"%@",typeClass);
        }
    }
    return self;
}

@end
```

打印结果:


![](http://upload-images.jianshu.io/upload_images/651640-c932469a15fdc790.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后进行下一步----**2.用该属性名作为键去字典中寻找对应的值.**



``` objective-c
 id value = [keyValues valueForKey:property.name];
 if (!value) continue;
```

接下来是第三步:**3.拿到值后将值转换为属性对应的类或类型.**

首先处理数字类型,如果模型的属性是数字类型,即`type.isNumberType == YES`.如果字典中的值是字符串类型的,需要将其转成`NSNumber`类型.如果本来就是基本数据类型,则不用进行任何转换.

``` objective-c
if (type.isNumberType){
    // 字符串->数字
    if ([value isKindOfClass:[NSString class]])
        value = [[[NSNumberFormatter alloc]init] numberFromString:value];
}
```

其中有一种情况,是需要进行特殊处理的.当模型的属性是`char`类型或者`bool`类型时,获取到的编码都为`c`,`char`对应`_numberType`,而`bool`对应`_boolType`.也就是说,`char`包含了`bool`的情况,`_numberType == YES`包含了`_boolType == YES`的情况.

``` objective-c
if (type.isNumberType){
            NSString *oldValue = value;
            // 字符串->数字
            if ([value isKindOfClass:[NSString class]]){
                value = [[[NSNumberFormatter alloc] init] numberFromString:value];
                if (type.isBoolType) {
                    NSString *lower = [oldValue lowercaseString];
                    if ([lower isEqualToString:@"yes"] || [lower isEqualToString:@"true"] ) {
                        value = @YES;
                    } else if ([lower isEqualToString:@"no"] || [lower isEqualToString:@"false"]) {
                        value = @NO;
                    }
                }
            }
        }
```

然后处理其他类型转成字符串类型的情况.

``` objective-c
else{
            if (typeClass == [NSString class]) {
                if ([value isKindOfClass:[NSNumber class]]) {
                    if (type.isNumberType)
                        // NSNumber -> NSString
                        value = [value description];
                }else if ([value isKindOfClass:[NSURL class]]){
                    // NSURL -> NSString
                    value = [value absoluteString];
                }
            }
        }
```

最后,进行**赋值**.

``` objective-c
[self setValue:value forKey:property.name];
```

最简单的字典转模型大致完成了,当然,还有很多细节没有完善,但细节总是随着需求的不断变化而不断增加的.



## JSON字符串 -> 模型





定义一个JSON字符串转成模型:

``` objective-c
/**
 *  JSON字符串 -> 模型
 */
void keyValues2object1()
{
    // 1.定义一个JSON字符串
    NSString *jsonString = @"{\"name\":\"Jack\", \"icon\":\"lufy.png\", \"age\":20}";

    // 2.将JSON字符串转为User模型
    User *user = [User objectWithKeyValues:jsonString];

    // 3.打印User模型的属性
    NSLog(@"name=%@, icon=%@, age=%d", user.name, user.icon, user.age);
}
```

这时程序会崩溃,因为没有对程序原来只对字典类型作处理:

``` objective-c
// 如果是字符串,到这行就崩了
id value = [keyValues valueForKey:property.name];
```

所以在这之前需要将JSON转成`Foundation`框架中的对象,苹果提供了强大的`NSJSONSerialization`.利用它,在刚开始传入字典/JSON字符串的时候将其进行转换.

``` objective-c
- (instancetype)setKeyValues:(id)keyValues{
    keyValues = [keyValues JSONObject];
......
}
```

该方法的具体实现如下,如果是`NSString`,就要先转成`NSData`再进行序列化.

``` objective-c
- (id)JSONObject{
    id foundationObj;
    if ([self isKindOfClass:[NSString class]]) {
        foundationObj = [NSJSONSerialization JSONObjectWithData:[(NSString *)self dataUsingEncoding:NSUTF8StringEncoding] options:kNilOptions error:nil];
    }else if ([self isKindOfClass:[NSData class]]){
        foundationObj = [NSJSONSerialization JSONObjectWithData:(NSData *)self options:kNilOptions error:nil];
    }
    return foundationObj?:self;
}
```



## 复杂的字典 -> 模型



定义一个模型中包含模型的复杂字典:

``` objective-c
NSDictionary *dict = @{
                           @"text" : @"是啊，今天天气确实不错！",

                           @"user" : @{
                                   @"name" : @"Jack",
                                   @"icon" : @"lufy.png"
                                   },

                           @"retweetedStatus" : @{
                                   @"text" : @"今天天气真不错！",

                                   @"user" : @{
                                           @"name" : @"Rose",
                                           @"icon" : @"nami.png"
                                           }
                                   }
                           };
```



对待这种字典的思路,类似于递归,当碰到模型中的属性类型是一个模型类时,将字典中的`值(Value)`作为字典处理.然后再调用字典转模型的方法返回一个模型类.所以在包装类型时还要有个属性表示它是否是自定义的模型类.判断的方法是看它是否是来自于`Foundation框架`的类.

``` objective-c
/** 类型是否来自于Foundation框架，比如NSString、NSArray */
@property (nonatomic, readonly, getter = isFromFoundation) BOOL fromFoundation;
```

在提取类型的方法中添加这样一条:

``` objective-c
else if (code.length > 3 && [code hasPrefix:@"@\""]) {
        // 去掉@"和"，截取中间的类型名称
        _code = [code substringWithRange:NSMakeRange(2, code.length - 3)];
        _typeClass = NSClassFromString(_code);
        _numberType = (_typeClass == [NSNumber class] || [_typeClass isSubclassOfClass:[NSNumber class]]);
  		// 判断是否是模型类
        _fromFoundation = [NSObject isClassFromFoundation:_typeClass];
  }
```

怎么判断是否来自`Foundation框架`呢? 下图展示了`Foundation框架(NSObject部分)`下的类结构.

 
![](http://upload-images.jianshu.io/upload_images/651640-7df1eb092bbf6ea2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

用一个`NSSet`(比用`NSArray`检索效率更高),返回一些常用基本的`Foundation框架`下继承自`NSObject`的类.

``` objective-c
static NSSet *foundationClasses_;

+ (NSSet *)foundationClasses
{
    if (foundationClasses_ == nil) {

        foundationClasses_ = [NSSet setWithObjects:
                              [NSURL class],
                              [NSDate class],
                              [NSValue class],
                              [NSData class],
                              [NSArray class],
                              [NSDictionary class],
                              [NSString class],
                              [NSAttributedString class], nil];
    }
    return foundationClasses_;
}
```

具体`isClassFromFoundation`的逻辑由类方法实现,在上面的集合中遍历.由于几乎所有类都是继承自`NSObject`,所以`NSObject`不能写入上面的集合当中,需要额外判断:

``` objective-c
+ (BOOL)isClassFromFoundation:(Class)c{
    if (c == [NSObject class]) return YES;
    __block BOOL result = NO;
    [[self foundationClasses] enumerateObjectsUsingBlock:^(Class foundationClass, BOOL *stop) {
        if ([c isSubclassOfClass:foundationClass]) {
            result = YES;
            *stop = YES;
        }
    }];
    return result;
}
```

得到结果后,需要在`setKeyValues:keyValues`这一核心方法中添加是否为模型类的判断:

``` objective-c
// 如果不是来自foundation框架的类并且不是基本数据类型 ,则递归
 if (!type.isFromFoundation && typeClass) {
 	value = [typeClass objectWithKeyValues:value];
 }
```



## 字典数组 -> 模型

稍复杂的一种情况是字典里装有数组的情况.

``` objective-c
NSDictionary *dict = @{
                           @"statuses" : @[
                                   @{
                                       @"text" : @"今天天气真不错！",

                                       @"user" : @{
                                               @"name" : @"Rose",
                                               @"icon" : @"nami.png"
                                               }
                                       },

                                   @{
                                       @"text" : @"明天去旅游了",

                                       @"user" : @{
                                               @"name" : @"Jack",
                                               @"icon" : @"lufy.png"
                                               }
                                       }

                                   ],

                           @"ads" : @[
                                   @{
                                       @"image" : @"ad01.png",
                                       @"url" : @"http://www.小码哥ad01.com"
                                       },
                                   @{
                                       @"image" : @"ad02.png",
                                       @"url" : @"http://www.小码哥ad02.com"
                                       }
                                   ],

                           @"totalNumber" : @"2014",
                           @"previousCursor" : @"13476589",
                           @"nextCursor" : @"13476599"
                           };
```

上面定义了一个字典,模型`StatusResult`有两个数组属性.

``` objective-c
@interface StatusResult : BaseObject

/** 存放着某一页微博数据（里面都是Status模型） */
@property (strong, nonatomic) NSMutableArray *statuses;
/** 存放着一堆的广告数据（里面都是Ad模型） */
@property (strong, nonatomic) NSArray *ads;
/** 总数 */
@property (strong, nonatomic) NSNumber *totalNumber;
/** 上一页的游标 */
@property (assign, nonatomic) long long previousCursor;
/** 下一页的游标 */
@property (assign, nonatomic) long long nextCursor;

@end
```

对于一个数组来说,你必须要告诉方法里面装的是什么模型,才能将字典中值为数组的成员转成模型.

在`MJExtension`中,提供了两种方式进行处理.

方式一,调用`NSObject`分类中得类方法:

``` objective-c
[StatusResult setupObjectClassInArray:^NSDictionary *{
        return @{
                 @"statuses" : @"Status",
                 // 或者 @"statuses" : [Status class],
                 @"ads" : @"Ad"
                 // 或者 @"ads" : [Ad class]
                 };
    }];
```

方式二,在模型的`.m`文件中实现方法供回调:

``` objective-c
+ (NSDictionary *)objectClassInArray
{
    return @{
             @"statuses" : @"Status",
     		 // 或者 @"statuses" : [Status class],
             @"ads" : @"Ad"
			 // 或者 @"ads" : [Ad class]
             };
}
```

原理上都差不多,都是通过代码进行回调,这个主要实现方式二.

在分类中声明一个`protocol`提供接口供模型类调用.

``` objective-c
@protocol MJKeyValue <NSObject>

+ (NSDictionary *) objectClassInArray;

@end
```

在转换的代码中设置添加设置数组模型的方法:

``` objective-c
if (!type.isFromFoundation && typeClass) {
    value = [typeClass objectWithKeyValues:value];
}
// 看该类是否实现了objectClassInArray方法
else if ([self.class respondsToSelector:@selector(objectClassInArray)]){
    id objectClass;
    // 如果是class类型,例如@"statuses" : [Status class]
  	objectClass = [self.class objectClassInArray][property.name];

    // 如果是NSString类型,例如@"statuses" : @"Status"
    if ([objectClass isKindOfClass:[NSString class]]) {
        objectClass = NSClassFromString(objectClass);
    }

	// 如果有值
    if (objectClass) {
        // 返回一个装了模型的数组
		value = [objectClass objectArrayWithKeyValuesArray:value];
    }

}
```

这时返回的值当然是个装满模型的数组模型.思路也很简单,对数组里的每一个成员都进行字典转模型的方法.如果其中的成员不是自定义模型类,那么直接返回.

``` objective-c
+ (NSMutableArray *)objectArrayWithKeyValuesArray:(id)keyValuesArray{

    if ([self isClassFromFoundation:self])
        return keyValuesArray;
    // 如果是json字符串,转成字典
    keyValuesArray = [keyValuesArray JSONObject];

    NSMutableArray *modelArray = [NSMutableArray array];

    // 遍历
    for (NSDictionary *keyValues in keyValuesArray) {
        // 对其中的模型调用字典转模型方法,并添加到数组中返回
        id model;
        model = [self objectWithKeyValues:keyValues];
        if (model) {
            [modelArray addObject:model];
        }
    }

    return modelArray;

}
```

------

## key的替换

``` objective-c
@interface IDAndDescription : NSObject

@property (nonatomic, copy) NSString *ID;
@property (nonatomic, copy) NSString *Description;

@end
```

实际开发中,服务器通常返回一个字段名为`id`,或者`description`的`JSON`数据,而这两个名字在OC中有特殊含义,如上所示,在定义属性的时候并不能使用这类名称.这时属性名与字典key不再是直接对应的关系,需要加入一层转换.

源码中key的替换也有几种方式选择,这里实现`replacedKeyFromPropertyName`这一方式.

过程是在要替换key的模型类中实现`replacedKeyFromPropertyName`方法,返回一个原始key和更名的key对应的字典.`replacedKeyFromPropertyName`在`protocol`中声明.

**实际上,也就是创建了一个方法来获取属性名与字典key的对应关系**.

在模型类中实现接口中的方法告知对应关系.

``` objective-c
@implementation IDAndDescription

+ (NSDictionary *)replacedKeyFromPropertyName{
    return @{
             @"ID" : @"id",
             @"Description" : @"description"
             };
}

@end
```

该方法从字典中需找要替换的key,参数是`property`的名字.如果字典中找不到对应的属性名,则不需要进行转换.

``` objective-c
+ (NSString *)propertyKey:(NSString *)propertyName{
    NSString *key;
    if ([self respondsToSelector:@selector(replacedKeyFromPropertyName)]) {
        key = [self replacedKeyFromPropertyName][propertyName];
    }
    return key?:propertyName;
}
```

在获取`值(value)`的时候,要将key替换成对应的key.

``` objective-c
id value = [keyValues valueForKey:[self.class propertyKey:property.name]];
if (!value) continue;
```

转换完成.

------

## 性能优化

将5个字典转模型的例子同时进行运行,在`+ properties`方法中添加一句打印.另外之前的例子都是有内存泄露的,这里添加了`free(properties)`修复了这个问题.

``` objective-c
+ (NSArray *)properties{
    NSLog(@"%@调用了properties方法",[self class]);
    NSMutableArray *propertiesArray = [NSMutableArray array];
    // 1.获得所有的属性
    unsigned int outCount = 0;
    objc_property_t *properties = class_copyPropertyList(self, &outCount);

    for (int i = 0; i < outCount; i++) {
        objc_property_t property = properties[i];

        MJProperty *propertyObj = [MJProperty propertyWithProperty:property];

        [propertiesArray addObject:propertyObj];
    }

    free(properties);

    return propertiesArray;
}
```

输出台输出如下:


![](http://upload-images.jianshu.io/upload_images/651640-8f912a614b68fbf0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到,很多的类都不止一次调用了获取属性的方法,对于一个类来说,要获取它的全部属性,只要获取一次就够了.获取到后将结果缓存起来,下次就不必进行不必要的计算.

> 注意:由于我写文章时手上的这份源码相对较早,缓存属性列表是通过一个全局字典来缓存的,而在最新版本的`MJExtension`中,已经换成了关联对象来实现.由于实现思路大致都是一样,并且效果相同,所以这里并不纠结用哪种方式.

``` objective-c
// 设置一个全局字典用来将类的属性都缓存起来
static NSMutableDictionary *cachedProperties_;
+ (void)load
{
    cachedProperties_ = [NSMutableDictionary dictionary];
}
```

将方法改写为:

``` objective-c
+ (NSArray *)properties
{
    NSMutableArray *cachedProperties = cachedProperties_[NSStringFromClass(self)];

    if (!cachedProperties) {

        NSLog(@"%@调用了properties方法",[self class]);

        cachedProperties = [NSMutableArray array];
        // 1.获得所有的属性
        unsigned int outCount = 0;
        objc_property_t *properties = class_copyPropertyList(self, &outCount);

        for (int i = 0; i < outCount; i++) {
            objc_property_t property = properties[i];

            MJProperty *propertyObj = [MJProperty propertyWithProperty:property];

            [cachedProperties addObject:propertyObj];
        }

        free(properties);

        cachedProperties_[NSStringFromClass(self)] = cachedProperties;
    }

    return cachedProperties;
}
```

此时控制台输出:


![](http://upload-images.jianshu.io/upload_images/651640-c02f07e20016b9f7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看每个类只经过一次获取全部属性.

------

除了缓存属性外,提取类型编码的过程也可以进一步缓存优化性能.

在下面的方法中加上一句打印:

``` objective-c
- (void)getTypeCode:(NSString *)code
{
    NSLog(@"%@",code);
  ......
}
```

控制台输出:


![](http://upload-images.jianshu.io/upload_images/651640-9587e3d51a08d4ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到一些常用的类型例如`NSString`多次调用了该方法.提取类型时,只要知道类名(在这里也就是`typeCode`),一个`MJPropertyType`就已经可以确定了.

重写了`- initWithTypeString:`方法:

``` objective-c
static NSMutableDictionary *cachedTypes_;
+ (void)load
{
    cachedTypes_ = [NSMutableDictionary dictionary];
}

+ (instancetype)propertyTypeWithAttributeString:(NSString *)string{
    return [[MJPropertyType alloc] initWithTypeString:string];
}

- (instancetype)initWithTypeString:(NSString *)string
{
    NSUInteger loc = 1;
    NSUInteger len = [string rangeOfString:@","].location - loc;
    NSString *typeCode = [string substringWithRange:NSMakeRange(loc, len)];

    if (!cachedTypes_[typeCode])
    {
        NSLog(@"%@",typeCode);
        self = [super init];
        [self getTypeCode:typeCode];
        cachedTypes_[typeCode] = self;
    }
    return self;
}
```

输出结果:



![](http://upload-images.jianshu.io/upload_images/651640-c2de638def5e4fcf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)




