# asm-doc-cn
asm的中文翻译文档，英文原版地址https://asm.ow2.io/asm4-guide.pdf

# 2.第二章 类
本章将解释如何使用核心 ASM API 生成和转换已编译的 Java 类。 首先会介绍已编译的类的组成，然后介绍相应的 ASM 接口、组件和工具来生成和转换它们，并附有许多说明性示例。 方法、注解和泛型的内容在接下来的章节中解释。
## 2.1 结构
### 2.1.1 概览
编译后的类整体结构非常简单。 实际上，与本机编译的应用程序不同，编译后的类保留了源代码中的结构信息和几乎所有符号。 事实上，编译后的类包含：
- 描述修饰符（例如公共或私有）、名称、超类、接口和类的注释的部分。
- 类中声明的每个字段都是一个section。 每个section都描述了字段的修饰符、名称、类型和注释。
- 类中声明的每个方法和构造函数也是一个section。 每个section都描述了修饰符、名称、返回和参数类型以及方法的注释。 它还包含方法的编译代码，以 Java 字节码指令序列的形式。
然而，源类和编译类之间存在一些差异：
- 一个编译的类只描述一个类，而一个源文件可以包含多个类。 例如，一个描述具有一个内部类的源文件被编译为两个类文件：一个用于主类，一个用于内部类。 然而，主类文件包含对其内部类的引用，而在方法内部定义的内部类包含对其封闭方法的引用。
- 编译后的类不含注释，但包含类型、字段、方法和代码属性（这些属性可用于将附加信息与这些元素相关联）。 从 Java 5 中引入用于相同目的注解以来，属性几乎变得毫无用处。
- 编译后的类不含包和导入部分，因此所有类型名称都必须是完全限定的。
另一个非常重要的结构差异是编译后的类包含一个常量池部分。 这个池是一个数组，包含出现在类中的所有数字、字符串和类型常量。 这些常量仅在常量池中定义一次，并且在类文件中所有用到的地方，持有它们的索引引用。 ASM 隐藏了与常量池相关的所有细节，这样您就不必费心了。 图 2.1 总结了编译类的整体结构。 Java 虚拟机规范的第 4 节中描述了确切的结构。

![图 2.1.：编译类的整体结构（* 表示零个或多个）](https://user-images.githubusercontent.com/25916578/148519943-2798f323-a562-4ddd-809e-ce026c1334de.jpeg)

编译后的类和源类另一个重要的区别是 Java 类型，下面的部分将会做下解释。
### 2.1.2. 内部名
很多情况下，类型被限制为类或接口。 比如一个类的超类，一个类实现的接口，或者一个方法抛出的异常，不能是原始类型或数组类型，而必须是类或接口类型。这些类型在编译后的类中是以内部名称形式存在的（类的内部名称即此类的完全限定名称，其中点替换为斜线）， 例如，String 的内部名称是 java/lang/String。
### 2.1.3. 类型描述符
内部名称仅用于限制为类或接口类型的类型。 Java 类型在带有类型描述符的编译类中表示（参见图 2.2）。

|Java type | Type descriptor|
| --- |--|
|boolean | Z|
|char | C|
|byte | B|
|short | S|
|int | I|
|float | F|
|long | J|
|double | D|
|Object | Ljava/lang/Object;|
|int[] | [I
|Object[][] | [[Ljava/lang/Object;|

原始类型的描述符是单个字符：Z 代表布尔值，C为字符，B 为字节，S 为短，I 为整数，F 为浮点数，J 为长和 D为双。 一个类类型的描述符是这个类的内部名称，前面是L，后面是分号。 
例如类型描述符的字符串是 Ljava/lang/String;。 最后一个数组类型的描述符是一个方括号，后跟数组元素类型的描述符。

### 2.1.4. 方法描述符
翻译结果
方法描述符是一个类型描述符列表，用于在单个字符串中描述方法的参数类型和返回类型。 方法描述符以左括号开头，后跟每个形参的类型描述符，后跟右括号，后跟返回类型的类型描述符，如果方法返回 void（方法描述符不包含 方法的名称或参数名称）。 

|Method declaration in source file | Method descriptor|
|---|---|
|void m(int i, float f) |(IF)V |
|int m(Object o) | (Ljava/lang/Object;)I|
|int[] m(int i, String s) | (ILjava/lang/String;)[I |
|Object m(int[] i) | ([I)Ljava/lang/Object;|

一旦你知道类型描述符是如何工作的，理解方法描述符就很容易了。 例如 (I)I 描述了一个方法，它接受一个 int 类型的参数，并返回一个 int。 上图给出了几个方法描述符的例子。

## 2.2. 接口和组件
### 2.2.1. 呈现方式
用于生成和转换已编译类的 ASM API 基于ClassVisitor 抽象类（见图 2.4）。 这个类中的每个方法对应同名的类文件结构部分（见图2.1)。 使用简单方法访问简单部分，该方法的入参见其方法描述。 通过返回辅助访问者类的方式，可以访问具有任意长度和复杂性的部分。 这是visitAnnotation、visitField 和visitMethod 方法的情况，它们分别返回AnnotationVisitor、FieldVisitor 和MethodVisitor。
```
public abstract class ClassVisitor {
    public ClassVisitor(int api);
    public ClassVisitor(int api, ClassVisitor cv);
    public void visit(int version, int access, String name, String signature, String superName, String[] interfaces);
    public void visitSource(String source, String debug);
    public void visitOuterClass(String owner, String name, String desc);
    AnnotationVisitor visitAnnotation(String desc, boolean visible);
    public void visitAttribute(Attribute attr);
    public void visitInnerClass(String name, String outerName, String innerName, int access);
    public FieldVisitor visitField(int access, String name, String desc, String signature, Object value);
    public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions);
    void visitEnd();
}
```

这些辅助类递归地使用相同的原则。 例如FieldVisitor抽象类中的每个方法对应同名的类文件子结构(如下图代码)，visitAnnotation返回一个辅助AnnotationVisitor，如ClassVisitor。 这些辅助访问者的创建和使用将在接下来的章节中解释：实际上本章仅限于可以单独使用 ClassVisitor 类解决的简单问题。 
```
public abstract class FieldVisitor {
    public FieldVisitor(int api);
    public FieldVisitor(int api, FieldVisitor fv);
    public AnnotationVisitor visitAnnotation(String desc, boolean visible);
    public void visitAttribute(Attribute attr);
    public void visitEnd();
}

```
