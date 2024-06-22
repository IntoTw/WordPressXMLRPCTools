---
title: 'Java编译期注解处理器详细使用方法'
date: 2020-10-15T00:00:00+08:00
lastmod: 2020-10-15T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["java","apt"]
categories: ["技术"]
lightgallery: true
---



## Java编译期注解处理器
Java编译期注解处理器，Annotation Processing Tool，简称APT，是Java提供给开发者的用于在编译期对注解进行处理的一系列API，这类API的使用被广泛的用于各种框架，如dubbo,lombok等。

Java的注解处理一般分为2种，最常见也是最显式化的就是Spring以及Spring Boot的注解实现了，在运行期容器启动时，根据注解扫描类，并加载到Spring容器中。而另一种就是本文主要介绍的注解处理，即编译期注解处理器，用于在编译期通过JDK提供的API，对Java文件编译前生成的Java语法树进行处理，实现想要的功能。

前段公司要求将原有dubbo迁入spring cloud架构，理所当然的最简单的方式，就是将原有的dubboRpc服务类，外面封装一层controller，并且将调用改成feignClient，这样能短时间的兼容原有其他未升级云模块的dubbo调用，之前考虑过其他方案，比如spring cloud sidecar。但是运维组反对，不建议每台机器多加一个服务，并且只是为了短时间过渡，没必要多加一个技术栈，所以考虑使用编译期处理器来快速生成类似的java代码，避免手动大量处理会产生操作失误。

练手项目示例的git源码：https://github.com/IntoTw/mob

## 启用注解处理器
增加这么一个类，实现AbstractProcessor的方法
```java
//注解处理器会扫描的包名
@SupportedAnnotationTypes("cn.intotw.*")
@SupportedSourceVersion(SourceVersion.RELEASE_8)
public class ModCloudAnnotationProcessor extends AbstractProcessor {
    private Messager messager;
    private JavacTrees trees;
    private TreeMaker treeMaker;
    private Names names;
    Map<String, JCTree.JCAssign> consumerSourceAnnotationValue=new HashMap<>();
    Map<String, JCTree.JCAssign> providerSourceAnnotationValue=new HashMap<>();
    java.util.List<String> javaBaseVarType;
    @Override
    public void init(ProcessingEnvironment processingEnv) {
        //基本构建，主要是初始化一些操作语法树需要的对象
        super.init(processingEnv);
        this.messager = processingEnv.getMessager();
        this.trees = JavacTrees.instance(processingEnv);
        Context context = ((JavacProcessingEnvironment) processingEnv).getContext();
        this.treeMaker = TreeMaker.instance(context);
        this.names = Names.instance(context);
    }
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (roundEnv.processingOver()) {
            return false;
        }
        //获取所有增加了自定义注解的element集合
        Set<? extends Element> set = roundEnv.getElementsAnnotatedWith(MobCloudConsumer.class);
        //遍历这个集合，这个集合的每个element相当于一个拥有自定义注解的需要处理的类。
        set.forEach(element -> {
            //获取语法树
            JCTree jcTree=trees.getTree(element);
            printLog("result :{}",jcTree);
        });
        return true;
    }
```
上面代码中获取的jctree就是那个class文件解析后的java语法树了，下面来看下有哪些操作。
## 遍历语法树
java语法树的遍历，并不是能像寻常树节点一样提供child之类的节点，而是通过TreeTranslator这个访问类的实现来做到的，这个类的可供实现的方法有很多，可以用来遍历语法树的注解、方法、变量，基本上语法树的所有java元素，都可以使用这个访问器来访问
```java
//获取源注解的参数
jcTree.accept(new TreeTranslator(){
    @Override
    public void visitAnnotation(JCTree.JCAnnotation jcAnnotation) {
        JCTree.JCIdent jcIdent=(JCTree.JCIdent)jcAnnotation.getAnnotationType();
        if(jcIdent.name.contentEquals("MobCloudConsumer")){
            printLog("class Annotation arg process:{}",jcAnnotation.toString());
            jcAnnotation.args.forEach(e->{
                JCTree.JCAssign jcAssign=(JCTree.JCAssign)e ;
                JCTree.JCIdent value = treeMaker.Ident(names.fromString("value"));
                JCTree.JCAssign targetArg=treeMaker.Assign(value,jcAssign.rhs);
                consumerSourceAnnotationValue.put(jcAssign.lhs.toString(),targetArg);
            });
        }
        printLog("获取参数如下:",consumerSourceAnnotationValue);
        super.visitAnnotation(jcAnnotation);
    }
});
```
## 语法树中的源节点
前面说了语法树是有一个个对象组成的，这些对象构成了语法树的一个个源节点，源节点对应java语法中核心的那些语法：
| 语法树节点类   | 具体对应的语法元素 |
| -------------- | ------------------ |
| JCClassDecl    | 类的定义           |
| JCMethodDecl   | 方法的定义         |
| JCAssign       | 等式（赋值）语句   |
| JCExpression   | 表达式             |
| JCAnnotation   | 注解               |
| JCVariableDecl | 变量定义           |
## 语法树节点的操作
既然说了语法树的那些重要节点，后面直接上案例，该如何操作。需要注意的一点是，Java语法树中所有的操作，对于语法树节点，都不能通过引用操作来复制，必须要从头到尾构造一个一模一样的对象并插入，否则编译是过不去的。
### 给类增加注解
```java
//该方法最后会给类新增一个@FeignClient(value="")的注解
private void addClassAnnotation(Element element) {
    JCTree jcTree = trees.getTree(element);
    jcTree.accept(new TreeTranslator(){
        //遍历所有类定义
        @Override
        public void visitClassDef(JCTree.JCClassDecl jcClassDecl) {
            JCTree.JCExpression arg;
            //创建一个value的赋值语句，作为注解的参数
            if(jcAssigns==null || jcAssigns.size()==0){
                arg=makeArg("value","");
            }
            printLog("jcAssigns :{}",jcAssigns);
            //创建注解对象
            JCTree.JCAnnotation jcAnnotation=makeAnnotation(PackageSupportEnum.FeignClient.toString(),List.of(arg));
            printLog("class Annotation add:{}",jcAnnotation.toString());
            //在原有类定义中append新的注解对象
            jcClassDecl.mods.annotations=jcClassDecl.mods.annotations.append(jcAnnotation);
            jcClassDecl.mods.annotations.forEach(e -> {
                printLog("class Annotation list:{}",e.toString());
            });
            super.visitClassDef(jcClassDecl);
        }
    });
}
public JCTree.JCExpression makeArg(String key,String value){
    //注解需要的参数是表达式，这里的实际实现为等式对象，Ident是值，Literal是value，最后结果为a=b
    JCTree.JCExpression arg = treeMaker.Assign(treeMaker.Ident(names.fromString(key)), treeMaker.Literal(value));
    return arg;
}
private JCTree.JCAnnotation makeAnnotation(String annotaionName, List<JCTree.JCExpression> args){
    JCTree.JCExpression expression=chainDots(annotaionName.split("\\."));
    JCTree.JCAnnotation jcAnnotation=treeMaker.Annotation(expression, args);
    return jcAnnotation;
}
```
### 给类增加import语句
```java
private void addImport(Element element,PackageSupportEnum... packageSupportEnums) {
    TreePath treePath = trees.getPath(element);
    JCTree.JCCompilationUnit jccu = (JCTree.JCCompilationUnit) treePath.getCompilationUnit();
    java.util.List<JCTree> trees = new ArrayList<>();
    trees.addAll(jccu.defs);
    java.util.List<JCTree> sourceImportList = new ArrayList<>();
    trees.forEach(e->{
        if(e.getKind().equals(Tree.Kind.IMPORT)){
            sourceImportList.add(e);
        }
    });
    java.util.List<JCTree.JCImport> needImportList=buildImportList(packageSupportEnums);
    for (int i = 0; i < needImportList.size(); i++) {
        boolean importExist=false;
        for (int j = 0; j < sourceImportList.size(); j++) {
            if(sourceImportList.get(j).toString().equals(needImportList.get(i).toString())){
                importExist=true;
            }
        }
        if(!importExist){
            trees.add(0,needImportList.get(i));
        }
    }
    printLog("import trees{}",trees.toString());
    jccu.defs=List.from(trees);
}
private java.util.List<JCTree.JCImport> buildImportList(PackageSupportEnum... packageSupportEnums) {
    java.util.List<JCTree.JCImport> targetImportList =new ArrayList<>();
    if(packageSupportEnums.length>0){
        for (int i = 0; i < packageSupportEnums.length; i++) {
            JCTree.JCImport needImport = buildImport(packageSupportEnums[i].getPackageName(),packageSupportEnums[i].getClassName());
            targetImportList.add(needImport);
        }
    }
    return targetImportList;
}
private JCTree.JCImport buildImport(String packageName, String className) {
    JCTree.JCIdent ident = treeMaker.Ident(names.fromString(packageName));
    JCTree.JCImport jcImport = treeMaker.Import(treeMaker.Select(
            ident, names.fromString(className)), false);
    printLog("add Import:{}",jcImport.toString());
    return jcImport;
}
```
### 构建一个内部类
这边演示了一个构建内部类的过程，基本就演示了深拷贝一个内部类的过程
```java
private JCTree.JCClassDecl buildInnerClass(JCTree.JCClassDecl sourceClassDecl, java.util.List<JCTree.JCMethodDecl> methodDecls) {
    java.util.List<JCTree.JCVariableDecl> jcVariableDeclList = buildInnerClassVar(sourceClassDecl);
    String lowerClassName=sourceClassDecl.getSimpleName().toString();
    lowerClassName=lowerClassName.substring(0,1).toLowerCase().concat(lowerClassName.substring(1));
    java.util.List<JCTree.JCMethodDecl> jcMethodDecls = buildInnerClassMethods(methodDecls,
            lowerClassName);
    java.util.List<JCTree> jcTrees=new ArrayList<>();
    jcTrees.addAll(jcVariableDeclList);
    jcTrees.addAll(jcMethodDecls);
    JCTree.JCClassDecl targetClassDecl = treeMaker.ClassDef(
            buildInnerClassAnnotation(),
            names.fromString(sourceClassDecl.name.toString().concat("InnerController")),
            List.nil(),
            null,
            List.nil(),
            List.from(jcTrees));
    return targetClassDecl;
}

private java.util.List<JCTree.JCVariableDecl> buildInnerClassVar(JCTree.JCClassDecl jcClassDecl) {
    String parentClassName=jcClassDecl.getSimpleName().toString();
    printLog("simpleClassName:{}",parentClassName);
    java.util.List<JCTree.JCVariableDecl> jcVariableDeclList=new ArrayList<>();
    java.util.List<JCTree.JCAnnotation> jcAnnotations=new ArrayList<>();
    JCTree.JCAnnotation jcAnnotation=makeAnnotation(PackageSupportEnum.Autowired.toString()
            ,List.nil());
    jcAnnotations.add(jcAnnotation);
    JCTree.JCVariableDecl jcVariableDecl = treeMaker.VarDef(treeMaker.Modifiers(1, from(jcAnnotations)),
            names.fromString(parentClassName.substring(0, 1).toLowerCase().concat(parentClassName.substring(1))),
            treeMaker.Ident(names.fromString(parentClassName)),
            null);
    jcVariableDeclList.add(jcVariableDecl);
    return jcVariableDeclList;
}

private JCTree.JCModifiers buildInnerClassAnnotation() {
    JCTree.JCExpression jcAssign=makeArg("value",providerSourceAnnotationValue.get("feignClientPrefix").rhs.toString().replace("\"",""));
    JCTree.JCAnnotation jcAnnotation=makeAnnotation(PackageSupportEnum.RequestMapping.toString(),
            List.of(jcAssign)
    );
    JCTree.JCAnnotation restController=makeAnnotation(PackageSupportEnum.RestController.toString(),List.nil());
    JCTree.JCModifiers mods=treeMaker.Modifiers(Flags.PUBLIC|Flags.STATIC,List.of(jcAnnotation).append(restController));
    return mods;
}
//深度拷贝内部类方法
private java.util.List<JCTree.JCMethodDecl> buildInnerClassMethods(java.util.List<JCTree.JCMethodDecl> methodDecls,String serviceName) {
    java.util.List<JCTree.JCMethodDecl> target = new ArrayList<>();
    methodDecls.forEach(e -> {
        if (!e.name.contentEquals("<init>")) {
            java.util.List<JCTree.JCVariableDecl> targetParams=new ArrayList<>();
            e.params.forEach(param->{
                JCTree.JCVariableDecl newParam=treeMaker.VarDef(
                        (JCTree.JCModifiers) param.mods.clone(),
                        param.name,
                        param.vartype,
                        param.init
                );
                printLog("copy of param:{}",newParam);
                targetParams.add(newParam);
            });
            JCTree.JCMethodDecl methodDecl = treeMaker.MethodDef(
                    (JCTree.JCModifiers) e.mods.clone(),
                    e.name,
                    (JCTree.JCExpression) e.restype.clone(),
                    e.typarams,
                    e.recvparam,
                    List.from(targetParams),
                    e.thrown,
                    treeMaker.Block(0L,List.nil()),
                    e.defaultValue
            );
            target.add(methodDecl);
        }
    });
    target.forEach(e -> {
        if (e.params.size() > 0) {
            for (int i = 0; i < e.params.size() ; i++) {
                JCTree.JCVariableDecl jcVariableDecl=e.params.get(i);
                if(i==0){
                    //第一个参数加requestbody注解，其他参数加requestparam注解，否则会报错
                    if(!isBaseVarType(jcVariableDecl.vartype.toString()))
                    {
                        jcVariableDecl.mods.annotations = jcVariableDecl.mods.annotations.append(makeAnnotation(PackageSupportEnum.RequestBody.toString(), List.nil()));
                    }else {
                        JCTree.JCAnnotation requestParam=makeAnnotation(PackageSupportEnum.RequestParam.toString(),
                                List.of(makeArg("value",jcVariableDecl.name.toString())));
                        jcVariableDecl.mods.annotations = jcVariableDecl.mods.annotations.append(requestParam);
                    }
                }else {
                    JCTree.JCAnnotation requestParam=makeAnnotation(PackageSupportEnum.RequestParam.toString(),
                            List.of(makeArg("value",jcVariableDecl.name.toString())));
                    jcVariableDecl.mods.annotations = jcVariableDecl.mods.annotations.append(requestParam);
                }

            }
        }
        printLog("sourceMethods: {}", e);
        //value
        JCTree.JCExpression jcAssign=makeArg("value","/"+e.name.toString());

        JCTree.JCAnnotation jcAnnotation = makeAnnotation(
                PackageSupportEnum.PostMapping.toString(),
                List.of(jcAssign)
        );
        printLog("annotation: {}", jcAnnotation);
        e.mods.annotations = e.mods.annotations.append(jcAnnotation);
        JCTree.JCExpressionStatement exec = getMethodInvocationStat(serviceName, e.name.toString(), e.params);
        if(!e.restype.toString().contains("void")){
            JCTree.JCReturn jcReturn=treeMaker.Return(exec.getExpression());
            e.body.stats = e.body.stats.append(jcReturn);
        }else {
            e.body.stats = e.body.stats.append(exec);
        }


    });
    return List.from(target);
}
//创建方法调用，如String.format()这种
private JCTree.JCExpressionStatement getMethodInvocationStat(String invokeFrom, String invokeMethod, List<JCTree.JCVariableDecl> args) {
    java.util.List<JCTree.JCIdent> params = new ArrayList<>();
    args.forEach(e -> {
        params.add(treeMaker.Ident(e.name));
    });
    JCTree.JCIdent invocationFrom = treeMaker.Ident(names.fromString(invokeFrom));
    JCTree.JCFieldAccess jcFieldAccess1 = treeMaker.Select(invocationFrom, names.fromString(invokeMethod));
    JCTree.JCMethodInvocation apply = treeMaker.Apply(nil(), jcFieldAccess1, List.from(params));
    JCTree.JCExpressionStatement exec = treeMaker.Exec(apply);
    printLog("method invoke:{}", exec);
    return exec;
}
```

## 使用方法
注解器的实际使用，需要在resource文件夹下的META-INF.services文件夹下，新建一个叫做javax.annotation.processing.Processor的文件，里面写上需要生效的类注解处理器的包名加类名，例如：cn.intotw.mob.ModCloudAnnotationProcessor。
然后如果是作为第三方jar包提供给别人，需要在maven打包时增加如下配置，主要也是把javax.annotation.processing.Processor文件也打包到对应目录：
```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <excludes>
                <exclude>META-INF/**/*</exclude>
            </excludes>
        </resource>
    </resources>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <version>2.6</version>
            <executions>
                <execution>
                    <id>process-META</id>
                    <phase>prepare-package</phase>
                    <goals>
                        <goal>copy-resources</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>target/classes</outputDirectory>
                        <resources>
                            <resource>
                                <directory>${basedir}/src/main/resources/</directory>
                                <includes>
                                    <include>**/*</include>
                                </includes>
                            </resource>
                        </resources>
                    </configuration>
                </execution>
            </executions>
        </plugin>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
        </plugin>
    </plugins>
</build>
```
### chainDots方法
```java
public  JCTree.JCExpression chainDots(String... elems) {
        assert elems != null;

        JCTree.JCExpression e = null;
        for (int i = 0 ; i < elems.length ; i++) {
            e = e == null ? treeMaker.Ident(names.fromString(elems[i])) : treeMaker.Select(e, names.fromString(elems[i]));
        }
        assert e != null;

        return e;
}
```
## 总结
建议这类复杂的语法树操作，不要用来直接在生产进行复杂的扩展，我们本次使用也只是为了快速生成代码。防止手动cp出现失误。原因还是maven构建过程很复杂，哪怕你本地测试通过，真的到了实际项目复杂的构建过程后，不一定能保证代码正确性，甚至会和dubbo以及lombok等组件冲突。

这个技术主要也是以摸索API使用为主，国内没有什么资料，国外的资料也都是语法和类的介绍，实际例子并不多，花了很多时间摸索具体使用的方法，基本能达到实现一切操作了，毕竟注解，方法，类，变量，方法调用，这些都能自定义了，基本也没有什么别的了。期间参考了不少lombok的源码，lombok是在java语法树节点之外利用自己的语法树节点封装了一层，简化和规范了很多操作，可惜我找了一下lombok貌似并没有提供类似于工具包辅助，所以更加深入的使用推荐参考lombok源码的实现。
