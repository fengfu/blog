---
title: Sonar自定义插件开发指南
date: 2017-11-29 17:13:16
tags: 杂谈
---
## 1 Sonar简介 ##

SonarQube是一个非常流行和强大的静态代码检查工具。它可以针对很多语言进行分析，比如Java、C#、JavaScript、PL/SQL等。SonarQube中默认提供了很多代码检查规则，但是有时候我们需要根据自己的需求编写规则，这篇文章会涵盖如何编写自定义规则的各个步骤。

## 2 环境准备 ##
 - JDK 1.8
 - Intellij Idea
 - Maven 3.x或更高
 - SonarQube 5.6
 - Sonar-runner-dist 2.4

## 3 开发步骤 ##

### 3.1 创建Plugin项目 ###

 首先创建一个SonarPlugin项目，建议从SonarSource官方github站点下载模板项目，这样不至于遗漏配置。下载的地址是：[下载链接](https://github.com/SonarSource/sonar-custom-plugin-example)

### 3.2 引入Pom依赖 ###
将模板项目中的pom文件修改为自己项目对应的名称，比如groupId、artifactId、version、name、description，其他的不必修改。

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.qunar.sonar</groupId>
    <artifactId>qunar-java</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>sonar-plugin</packaging>

    <name>Qunar SonarQube Java Rules</name>
    <description>Qunar Java Rules for SonarQube</description>
    <inceptionYear>2016</inceptionYear>

    <properties>
        <sonar.version>6.3</sonar.version> <!-- this 6.3 is only required to be compliant with SonarLint and it is required
			even if you just want to be compliant with SonarQube 5.6 -->
        <java.plugin.version>4.10.0.10260</java.plugin.version>
        <sslr.version>1.21</sslr.version>
        <gson.version>2.6.2</gson.version>
    </properties>

    <dependencies>
        <!--请不要调整下面三项依赖的顺序，避免查看依赖的类时无法查看源码-->
        <dependency>
            <groupId>org.sonarsource.sonarqube</groupId>
            <artifactId>sonar-plugin-api</artifactId>
            <version>${sonar.version}</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.sonarsource.java</groupId>
            <artifactId>java-frontend</artifactId>
            <version>${java.plugin.version}</version>
        </dependency>

        <dependency>
            <groupId>org.sonarsource.java</groupId>
            <artifactId>sonar-java-plugin</artifactId>
            <!--<type>sonar-plugin</type>-->
            <version>${java.plugin.version}</version>
            <scope>provided</scope>
        </dependency>
        <!--请不要调整上述三项依赖的顺序，避免查看依赖的类时无法查看源码-->

        <dependency>
            <groupId>org.sonarsource.sslr-squid-bridge</groupId>
            <artifactId>sslr-squid-bridge</artifactId>
            <version>2.6.1</version>
            <exclusions>
                <exclusion>
                    <groupId>org.codehaus.sonar.sslr</groupId>
                    <artifactId>sslr-core</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.codehaus.sonar</groupId>
                    <artifactId>sonar-plugin-api</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.codehaus.sonar.sslr</groupId>
                    <artifactId>sslr-xpath</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>jcl-over-slf4j</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>slf4j-api</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <dependency>
            <groupId>org.sonarsource.java</groupId>
            <artifactId>java-checks-testkit</artifactId>
            <version>${java.plugin.version}</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>com.google.code.gson</groupId>
            <artifactId>gson</artifactId>
            <version>${gson.version}</version>
        </dependency>

        <dependency>
            <groupId>org.sonarsource.sslr</groupId>
            <artifactId>sslr-testing-harness</artifactId>
            <version>${sslr.version}</version>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.6.2</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.11</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <version>3.6.2</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>0.9.30</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.sonarsource.sonar-packaging-maven-plugin</groupId>
                <artifactId>sonar-packaging-maven-plugin</artifactId>
                <version>1.17</version>
                <extensions>true</extensions>
                <configuration>
                    <pluginKey>qunar-java</pluginKey>
                    <pluginName>QunarJava</pluginName>
                    <pluginClass>com.qunar.sonar.java.JavaRulesPlugin</pluginClass>
                    <sonarLintSupported>true</sonarLintSupported>
                    <sonarQubeMinVersion>5.6</sonarQubeMinVersion> <!-- allow to depend on API 6.x but run on LTS -->
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.6.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>

            <!-- only required to run UT - these are UT dependencies -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <version>2.10</version>
                <executions>
                    <execution>
                        <id>copy</id>
                        <phase>test-compile</phase>
                        <goals>
                            <goal>copy</goal>
                        </goals>
                        <configuration>
                            <artifactItems>
                                <artifactItem>
                                    <groupId>org.apache.commons</groupId>
                                    <artifactId>commons-collections4</artifactId>
                                    <version>4.0</version>
                                    <type>jar</type>
                                </artifactItem>
                                <artifactItem>
                                    <groupId>javax</groupId>
                                    <artifactId>javaee-api</artifactId>
                                    <version>6.0</version>
                                </artifactItem>
                                <artifactItem>
                                    <groupId>org.springframework</groupId>
                                    <artifactId>spring-webmvc</artifactId>
                                    <version>4.3.3.RELEASE</version>
                                </artifactItem>
                                <artifactItem>
                                    <groupId>org.springframework</groupId>
                                    <artifactId>spring-webmvc</artifactId>
                                    <version>4.3.3.RELEASE</version>
                                </artifactItem>
                                <artifactItem>
                                    <groupId>org.springframework</groupId>
                                    <artifactId>spring-web</artifactId>
                                    <version>4.3.3.RELEASE</version>
                                </artifactItem>
                                <artifactItem>
                                    <groupId>org.springframework</groupId>
                                    <artifactId>spring-context</artifactId>
                                    <version>4.3.3.RELEASE</version>
                                </artifactItem>
                            </artifactItems>
                            <outputDirectory>${project.build.directory}/test-jars</outputDirectory>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>

```

### 3.3 编写具体的Rule ###

一个自定义规则的实现，首先在规则类前面使用@Rule注解声明规则的相关信息，比如key、name、description、priority、tag。其次要继承IssuableSubscriptionVisitor或者继承BaseTreeVisitor并实现JavaFileScanner接口，这两种实现方式存在一些差异，IssuableSubscriptionVisitor是一个抽象类，继承自SubscriptionVisitor，而SubscriptionVisitor则实现了JavaFileScanner接口；而BaseTreeVisitor则是实现了TreeVisitor接口。相比来说，JavaFileScanner更偏底层一些，直接是面向Java文件的分析，而TreeVisitor则是针对于Java文件的语法结构了，比如类、防止、赋值语句、catch块之类的了。一般情况下，推荐使用继承IssuableSubscriptionVisitor的方式，因为这个类的封装层次更高，更简便。在规则类中，你只需要重载nodesToVisit和visitNode方法就可以了。
nodesToVisit方法是说明该规则做检查的类型集合，比如METHOD_INVOCATION（方法调用）、CATCH（catch块）、MEMBER_SELECT（属性选择）等。比如下面的代码，就是对源码中的方法调用做规则检查。
visitNode是自定义规则中的核心部分，你所要实现的代码检查逻辑就是在这个方法中完成的。

```
import com.google.common.collect.ImmutableList;
import com.qunar.sonar.java.util.QualifiedNameUtil;
import org.sonar.check.Priority;
import org.sonar.check.Rule;
import org.sonar.plugins.java.api.IssuableSubscriptionVisitor;
import org.sonar.plugins.java.api.tree.MethodInvocationTree;
import org.sonar.plugins.java.api.tree.Tree;

import java.util.List;

/**
 * Created by fengfu on 2017/11/10.
 */
@Rule(
        key = "DisallowedFDUsageCheck",
        name = "Qunar Flight disallowed FD usage in InfoCenter check",
        description = "Some kinds of usage for FD are forbidden in Flight BU of Qunar.com.",
        priority = Priority.BLOCKER,
        tags = {"bug"}
)
public class DisallowedFDUsageCheck extends IssuableSubscriptionVisitor {
    private static final String[] FORBIDDEN_METHODS = {
            "Infocenter.reloadFDData", "Infocenter.getFliterDateFDData",
            "Infocenter.getFDData", "Infocenter.getValidFDPrice",
            "Infocenter.getFDPrice"
    };

    @Override
    public List<Tree.Kind> nodesToVisit() {
        return ImmutableList.of(Tree.Kind.METHOD_INVOCATION);
    }

    @Override
    public void visitNode(Tree tree) {
        MethodInvocationTree mTree = (MethodInvocationTree)tree;
        String methodName = QualifiedNameUtil.fullQualifiedName(mTree.methodSelect());

        if (isForbiddenMethod(methodName)){
            reportIssue(tree, methodName + "已被禁用，详情请查看Wiki:http://wiki.corp.qunar.com/confluence/pages/viewpage.action?pageId=182623492");
        }

        super.visitNode(tree);
    }

    private static boolean isForbiddenMethod(String methodName) {
        for (String name : FORBIDDEN_METHODS) {
            if (name.equals(methodName)) {
                return true;
            }
        }
        return false;
    }

    @Override
    public void leaveNode(Tree tree) {
        super.leaveNode(tree);
    }
}
```

当然如果一些复杂的业务规则，可能你不得不实现JavaFileScanner接口了。如下面的代码所示：

```
import com.qunar.sonar.java.util.QualifiedNameUtil;
import org.sonar.api.utils.log.Logger;
import org.sonar.api.utils.log.Loggers;
import org.sonar.check.Priority;
import org.sonar.check.Rule;
import org.sonar.plugins.java.api.JavaFileScanner;
import org.sonar.plugins.java.api.JavaFileScannerContext;
import org.sonar.plugins.java.api.tree.*;

/**
 * Created by fengfu on 2017/2/27.
 */
@Rule(
        key = "QunarFlightDisallowedClassCheck",
        name = "Qunar Flight disallowed class check",
        description = "Some classes are forbidden in Flight BU of Qunar.com.",
        priority = Priority.BLOCKER,
        tags = {"bug"}
)
public class DisallowedClassCheck extends BaseTreeVisitor implements JavaFileScanner {
    private static Logger logger = Loggers.get(DisallowedClassCheck.class);
    private final String IMPORT_NAME = "com.qunar.base.meerkat.monitor.QMonitor";

    public void scanFile(JavaFileScannerContext context) {
        logger.info("Start to scan file {}", context.getFile().getName());
        CompilationUnitTree cut = context.getTree();

        for (ImportClauseTree importClauseTree : cut.imports()){
            ImportTree importTree = null;
            if (importClauseTree.is(Tree.Kind.IMPORT)) {
                importTree = (ImportTree) importClauseTree;
            }

            if (importTree == null || importTree.isStatic()) {
                continue;
            }

            String importName = QualifiedNameUtil.fullQualifiedName(importTree.qualifiedIdentifier());

            if (IMPORT_NAME.equals(importName)){
                logger.error("Disallowed class {} found.", IMPORT_NAME);
                context.reportIssue(this, importTree, IMPORT_NAME + "已被禁用.");
            }
        }

        scan(cut);
    }
}
```

### 3.4 单元测试 ###

规则检查代码写完之后，就可以针对这个规则进行单元测试了。单元测试的方法比较简单，就是使用JUnit框架编写test case，针对特定的目标文件，调用Sonar提供的JavaCheckVerifier.verify方法进行测试，示例代码如下：

```
import com.qunar.flight.sonar.java.checks.DisallowedClassCheck;
import org.junit.Test;
import org.sonar.java.checks.verifier.JavaCheckVerifier;

public class DisallowedClassCheckTest {
    @Test
    public void test() {
        JavaCheckVerifier.verify("src/test/files/flight/DisallowedClassCase.java", new DisallowedClassCheck());
    }
}
```
如果目标文件存在问题，那么控制台会打印出问题在目标文件中的位置：
```
java.lang.AssertionError: Unexpected at [4]

	at org.sonar.java.checks.verifier.CheckVerifier.assertMultipleIssue(CheckVerifier.java:209)
	at org.sonar.java.checks.verifier.CheckVerifier.checkIssues(CheckVerifier.java:181)
	at org.sonar.java.checks.verifier.JavaCheckVerifier.scanFile(JavaCheckVerifier.java:274)
	at org.sonar.java.checks.verifier.JavaCheckVerifier.scanFile(JavaCheckVerifier.java:256)
	at org.sonar.java.checks.verifier.JavaCheckVerifier.scanFile(JavaCheckVerifier.java:222)
	at org.sonar.java.checks.verifier.JavaCheckVerifier.verify(JavaCheckVerifier.java:105)
	at com.qunar.flight.sonar.java.checks.test.DisallowedClassCheckTest.test(DisallowedClassCheckTest.java:10)
```
如果目标文件中不存在任何问题，会提示“java.lang.IllegalStateException: At least one issue expected”，意思就是目标文件中需要至少存在一个规则所要检查的问题。

### 3.5 编写RulesList ###

单元测试完成之后，就可以往RuleList的集合中中添加自定义规则规则了。这个RuleList只是一个简单的集合类，包含了所有你需要注册到Sonar中的规则：

```
import com.google.common.collect.ImmutableList;
import java.util.List;

import com.qunar.flight.sonar.java.checks.DisallowedClassCheck;
import com.qunar.flight.sonar.java.checks.DisallowedFDUsageCheck;
import com.qunar.flight.sonar.java.checks.GuavaCacheLoaderReturnNullCheck;
import org.sonar.plugins.java.api.JavaCheck;

public final class RulesList {

  private RulesList() {
  }

  public static List<Class> getChecks() {
    return ImmutableList.<Class>builder().addAll(getJavaChecks()).addAll(getJavaTestChecks()).build();
  }

    /**
     * 在此方法中添加需要注册的规则
     * @return
     */
  public static List<Class<? extends JavaCheck>> getJavaChecks() {
    return ImmutableList.<Class<? extends JavaCheck>>builder()
      .add(DisallowedClassCheck.class)
      .add(GuavaCacheLoaderReturnNullCheck.class)
      .add(DisallowedFDUsageCheck.class)

      .build();
  }

  public static List<Class<? extends JavaCheck>> getJavaTestChecks() {
    return ImmutableList.<Class<? extends JavaCheck>>builder()
      .build();
  }
}
```


### 3.6 编写CustomRulesDefinition ###

CustomRulesDefinition可以理解为自定义规则集的元数据，在这个类中，首先要声明自定义规则集的repository，另外还需要将相应的规则集加入到前面声明的repository中。

```
mport com.google.common.annotations.VisibleForTesting;
import com.google.common.collect.Iterables;
import com.google.common.io.Resources;
import com.google.gson.Gson;
import org.apache.commons.lang.StringUtils;
import org.sonar.api.rule.RuleStatus;
import org.sonar.api.rules.RuleType;
import org.sonar.api.server.debt.DebtRemediationFunction;
import org.sonar.api.server.rule.RulesDefinition;
import org.sonar.api.server.rule.RulesDefinitionAnnotationLoader;
import org.sonar.api.utils.AnnotationUtils;
import org.sonar.check.Cardinality;
import org.sonar.squidbridge.annotations.RuleTemplate;

import javax.annotation.Nullable;
import java.io.IOException;
import java.net.URL;
import java.nio.charset.StandardCharsets;
import org.sonar.plugins.java.Java;
import java.util.List;
import java.util.Locale;

/**
 * Declare rule metadata in server repository of rules.
 * That allows to list the rules in the page "Rules".
 */
public class JavaRulesDefinition implements RulesDefinition {

    // don't change that because the path is hard coded in CheckVerifier
    private static final String RESOURCE_BASE_PATH = "/org/sonar/l10n/java/rules/squid";

    public static final String REPOSITORY_KEY = "qunar-java";

    private final Gson gson = new Gson();

    @Override
    public void define(Context context) {
        NewRepository repository = context
                .createRepository(REPOSITORY_KEY, Java.KEY)//定义Repository key
                .setName("QunarJava");//定义Repository名称

        List<Class> checks = RulesList.getChecks();//获取自定义的check列表
        new RulesDefinitionAnnotationLoader().load(repository, Iterables.toArray(checks, Class.class));//获取注解信息

        for (Class ruleClass : checks) {
            newRule(ruleClass, repository);//将规则添加到Repository中
        }
        repository.done();
    }

    @VisibleForTesting
    protected void newRule(Class<?> ruleClass, NewRepository repository) {

        org.sonar.check.Rule ruleAnnotation = AnnotationUtils.getAnnotation(ruleClass, org.sonar.check.Rule.class);
        if (ruleAnnotation == null) {
            throw new IllegalArgumentException("No Rule annotation was found on " + ruleClass);
        }
        String ruleKey = ruleAnnotation.key();
        if (StringUtils.isEmpty(ruleKey)) {
            throw new IllegalArgumentException("No key is defined in Rule annotation of " + ruleClass);
        }
        NewRule rule = repository.rule(ruleKey);
        if (rule == null) {
            throw new IllegalStateException("No rule was created for " + ruleClass + " in " + repository.key());
        }
        ruleMetadata(ruleClass, rule);

        rule.setTemplate(AnnotationUtils.getAnnotation(ruleClass, RuleTemplate.class) != null);
        if (ruleAnnotation.cardinality() == Cardinality.MULTIPLE) {
            throw new IllegalArgumentException("Cardinality is not supported, use the RuleTemplate annotation instead for " + ruleClass);
        }
    }

    private String ruleMetadata(Class<?> ruleClass, NewRule rule) {
        String metadataKey = rule.key();
        org.sonar.java.RspecKey rspecKeyAnnotation = AnnotationUtils.getAnnotation(ruleClass, org.sonar.java.RspecKey.class);
        if (rspecKeyAnnotation != null) {
            metadataKey = rspecKeyAnnotation.value();
            rule.setInternalKey(metadataKey);
        }
        addHtmlDescription(rule, metadataKey);
        addMetadata(rule, metadataKey);
        return metadataKey;
    }

    private void addMetadata(NewRule rule, String metadataKey) {
        URL resource = JavaRulesDefinition.class.getResource(RESOURCE_BASE_PATH + "/" + metadataKey + "_java.json");
        if (resource != null) {
            RuleMetatada metatada = gson.fromJson(readResource(resource), RuleMetatada.class);
            rule.setSeverity(metatada.defaultSeverity.toUpperCase(Locale.US));
            rule.setName(metatada.title);
            rule.addTags(metatada.tags);
            rule.setType(RuleType.valueOf(metatada.type));
            rule.setStatus(RuleStatus.valueOf(metatada.status.toUpperCase(Locale.US)));
            if (metatada.remediation != null) {
                rule.setDebtRemediationFunction(metatada.remediation.remediationFunction(rule.debtRemediationFunctions()));
                rule.setGapDescription(metatada.remediation.linearDesc);
            }
        }
    }

    private static void addHtmlDescription(NewRule rule, String metadataKey) {
        URL resource = JavaRulesDefinition.class.getResource(RESOURCE_BASE_PATH + "/" + metadataKey + "_java.html");
        if (resource != null) {
            rule.setHtmlDescription(readResource(resource));
        }
    }

    private static String readResource(URL resource) {
        try {
            return Resources.toString(resource, StandardCharsets.UTF_8);
        } catch (IOException e) {
            throw new IllegalStateException("Failed to read: " + resource, e);
        }
    }

    private static class RuleMetatada {
        String title;
        String status;
        @Nullable
        Remediation remediation;

        String type;
        String[] tags;
        String defaultSeverity;
    }

    private static class Remediation {
        String func;
        String constantCost;
        String linearDesc;
        String linearOffset;
        String linearFactor;

        public DebtRemediationFunction remediationFunction(DebtRemediationFunctions drf) {
            if (func.startsWith("Constant")) {
                return drf.constantPerIssue(constantCost.replace("mn", "min"));
            }
            if ("Linear".equals(func)) {
                return drf.linear(linearFactor.replace("mn", "min"));
            }
            return drf.linearWithOffset(linearFactor.replace("mn", "min"), linearOffset.replace("mn", "min"));
        }
    }
}
```

### 3.7 编写CustomJavaFileCheckRegistrar ###

CustomJavaFileCheckRegistrar要实现CheckRegistrar接口。CustomJavaFileCheckRegistrar类的作用是注册代码扫描时需要的规则类。

```
import java.util.List;
import org.sonar.plugins.java.api.CheckRegistrar;
import org.sonar.plugins.java.api.JavaCheck;

/**
 * Provide the "checks" (implementations of rules) classes that are going be executed during
 * source code analysis.
 *
 * This class is a batch extension by implementing the {@link org.sonar.plugins.java.api.CheckRegistrar} interface.
 */
public class JavaFileCheckRegistrar implements CheckRegistrar {

  /**
   * Register the classes that will be used to instantiate checks during analysis.
   */
  @Override
  public void register(RegistrarContext registrarContext) {
    // Call to registerClassesForRepository to associate the classes with the correct repository key
    registrarContext.registerClassesForRepository(JavaRulesDefinition.REPOSITORY_KEY, checkClasses(), testCheckClasses());
  }

  /**
   * Lists all the main checks provided by the plugin
   */
  public static List<Class<? extends JavaCheck>> checkClasses() {
    return RulesList.getJavaChecks();//返回RulesList中定义的checks
  }

  /**
   * Lists all the test checks provided by the plugin
   */
  public static List<Class<? extends JavaCheck>> testCheckClasses() {
    return RulesList.getJavaTestChecks();//返回RulesList中定义的check classes
  }
}
```

### 3.8 编写CustomJavaRulesPlugin ###

这个类是Sonar插件的入口，它继承了org.sonar.api.SonarPlugin类，包含了SonarQube启动时需要实例化的server扩展（CustomRulesDefinition）和代码分析时需要实例化的批量扩展（JavaFileCheckRegistrar）。

这个类基本不需要修改（除包名外），按照以下内容copy一份即可。

```
import org.sonar.api.Plugin;

/**
 * Entry point of your plugin containing your custom rules
 */
public class JavaRulesPlugin implements Plugin {

  @Override
  public void define(Context context) {

    // server extensions -> objects are instantiated during server startup
    context.addExtension(CustomRulesDefinition.class);

    // batch extensions -> objects are instantiated during code analysis
    context.addExtension(CustomJavaFileCheckRegistrar.class);

  }
}
```

至此，一个Sonar自定义插件所需的代码工作就结束了。剩下的就是进行整合性的测试，毕竟单元测试通过不代表插件不会对现有系统造成影响，有时候插件中的错误会导致SonarQube启动失败的。

## 5 整合测试 ##

通过mvn package命令编译、打包，形成jar文件。将jar文件上传到SonarQube extensions/plugins目录下，重启SonarQube服务即可完成Sonar自定义规则插件的部署。部署结束，通过SonarQube bin目录下的脚本重启SonarQube，如果SonarQube启动成功，说明插件没问题，整合测试成功，你可以在sonar的rules模块中，搜索到自定义的规则了。

![](https://farm5.staticflickr.com/4566/38001442235_a562e60306.jpg)

## 6 启用规则 ##
插件部署到SonarQube之后，默认是不启用的，需要在SonarQube的管理界面中启用规则，才能让此规则在代码检查过程中生效。

![](https://farm5.staticflickr.com/4524/38888239671_35af64c4bf.jpg)

至此，Sonar插件开发的流程就介绍完了。如前面所言，SonarQube是个强大的代码检查工具，一篇文章是无法介绍完的，尤其Sonar的语法树部分，涵盖了各种语言各种语法的方方面面，大家可以通过Sonar的[官方文档](https://docs.sonarqube.org/display/SONAR/)了解更多的细节。
