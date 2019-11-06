---
title: kubernetes之Jenkins 流水线语法
subtitle: 基于kubernetes 的CI/CD
date: 2019-10-14
tags: ["kubernetes", "jenkins x", "devops","jx","github","Chartmuseum","Monocular","Nexus","Draft","skaffold","helm","jenkins"]
keywords: ["kubernetes", "ldap",  "更新", "kubectl", "docker"]
slug: kubernetes-jenkinsfile
gitcomment: false
bigimg: [{src: "/img/k8s/k8s001.jpg", desc: ""}]
category: "kubernetes"
---

声明式和脚本化的流水线从根本上是不同的。 声明式流水线的是 Jenkins 流水线更近的特性:

- 相比脚本化的流水线语法，它提供更丰富的语法特性,
- 是为了使编写和读取流水线代码更容易而设计的。

<!--more-->

## 字符串插值

Jenkins Pipeline使用与Groovy相同的语法进行字符串插值。Groovy的字符串插值支持可能会让很多语言新手感到困惑。虽然Groovy支持使用单引号或双引号来声明一个字符串，例如：
```Jenkinsfile
def singlyQuoted = 'Hello'
def doublyQuoted = "World"
```
只有后一个字符串将支持基于美元符号（$）的字符串插值，例如：
```Jenkinsfile
def username = 'Jenkins'
echo 'Hello Mr. ${username}'
echo "I said, Hello Mr. ${username}"
```

会导致：
```Jenkinsfile
Hello Mr. ${username}
I said, Hello Mr. Jenkins
```
## 使用环境变量

Jenkins管道通过全局变量显示环境变量，全局变量env可在任何位置使用Jenkinsfile。在Jenkins Pipeline中可访问的环境变量的完整列表记录在 localhost:8080/pipeline-syntax/globals#env中，假设Jenkins主要运行localhost:8080，并包括：

BUILD_ID 当前版本ID与Jenkins版本1.597 +中创建的BUILD_NUMBER相同

JOB_NAME 此版本的项目名称，例如“foo”或“foo / bar”。

引用或使用这些环境变量可以像访问Groovy Map中的任何键一样完成 ，例如：
```Jenkinsfile
Jenkinsfile（声明式管道）
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo "Running ${env.BUILD_ID} on ${env.JENKINS_URL}"
            }
        }
    }
}
```
```Jenkinsfile
env.MYTOOL_VERSION = '1.33'
node {
  sh '/usr/local/mytool-$MYTOOL_VERSION/bin/start'
}
``
设置环境变量

根据是使用声明式管道还是脚本式管道，在Jenkins管道中设置环境变量的方式会有所不同。

声明性管道支持环境 指令，而脚本管道的用户必须使用该withEnv步骤。
​```Jenkinsfile
Jenkinsfile（声明式管道）
pipeline {
    agent any
    environment {            ①
        CC = 'clang'
    }
    stages {
        stage('Example') {
            environment {               ②
                DEBUG_FLAGS = '-g'
            }
            steps {
                sh 'printenv'
            }
        }
    }
}
```
这里写图片描述
用于秘密文本，用户名和密码以及秘密文件

Jenkins的声明式Pipeline语法具有credentials()辅助方法（在environment指令中使用），该方法支持 秘密文本，用户名和密码以及秘密文件凭证。如果您想处理其他类型的凭证，请参阅对于其他凭证类型部分（如下）。

## 秘密文本
以下管道代码显示了如何使用环境变量创建管道以用于秘密文本凭证的示例。

在此示例中，将两个秘密文本凭证分配给单独的环境变量以访问Amazon Web Services（AWS）。这些凭证将在Jenkins中以其各自的凭证ID
jenkins-aws-secret-key-id和jenkins-aws-secret-access-key
```Jenkinsfile
Jenkinsfile（声明式管道）
pipeline {
    agent {
        // Define agent details here
    }
    environment {
        AWS_ACCESS_KEY_ID     = credentials('jenkins-aws-secret-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('jenkins-aws-secret-access-key')
    }
    stages {
        stage('Example stage 1') {
            steps {
                //        ①
            }
        }
        stage('Example stage 2') {
            steps {
                //                ②
            }
        }
    }
}
```
这里写图片描述
处理参数

声明式流水线支持即用即设的参数, 允许流水线通过参数在运行时接受用户指定的参数。使用脚本管道配置参数的properties步骤可以在代码片段生成器中找到。

如果您使用‘带参数的构建’选项将管道配置为接受参数， 则可以将这些参数作为params变量子类进行访问。

假设‘Greeting’的字符串参数已在配置中Jenkinsfile， 它可以通过${params.Greeting}来访问该参数
```Jenkinsfile
Jenkinsfile（声明式管道）
pipeline {
    agent any
    parameters {
        string(name: 'Greeting', defaultValue: 'Hello', description: 'How should I greet the world?')
    }
    stages {
        stage('Example') {
            steps {
                echo "${params.Greeting} World!"
            }
        }
    }
}
```

## 处理失败

声明式流水线通过其post部分支持健壮的失败处理，允许声明多个不同的“post条件”。 例如: always, unstable, success, failure和changed。管道语法部分提供了更多关于如何使用各种post条件的详细信息。
```Jenkinsfile
Jenkinsfile（声明式管道）
pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                sh 'make check'
            }
        }
    }
    post {
        always {
            junit '**/target/*.xml'
        }
        failure {
            mail to: team@example.com, subject: 'The Pipeline failed :('
        }
    }
}
```

## agent

该agent部分指定整个管道或特定阶段将在Jenkins环境中执行的位置，具体取决于该agent部分的放置位置。该部分必须在pipeline块的顶层定义 ，但stage级别使用是可选的。
说明 	选项
必须 	是
参数 	详情如下
允许 	在顶层管道块和每个stage块中
参数

为了支持广泛的用例对管道作者来说，agent部分支持一些不同类型的参数。这些参数可以应用于管道块的顶层，也可以应用于每个stage指令。
any

在任何可用的agent上执行管道或stage。例如:agent any
none

当在管道块的顶层应用时，不会为整个管道运行分配全局代理，每个stage都需要包含它自己的agent部分。例如:agent none
常用选项

这些选项可以应用两个或多个agent实现。除非明确说明，否则不需要它们。
label

一个字符串。用于运行管道或单个stage的标签。

此选项对于node、docker和dockerfile是有效的，并且对于node是必需的。
customWorkspace

一个字符串。运行管道或单独的stage，该agent应用于这个自定义工作区，而不是默认的。它可以是相对路径，在这种情况下，自定义工作区将位于node的工作区根之下，或者是绝对路径。例如:
```Jenkinsfile
agent {
    node {
        label 'my-defined-label'
        customWorkspace '/some/other/path'
    }
}
```

这个选项对于docker和dockerfile是有效的，并且只有在单独的stage使用agent时才有效果。
post

post部分定义了一个或多个附加步骤，这些步骤在管道或stage的运行完成时运行(取决于管道内的post部分的位置)。post可以支持下列后置条件块:always、changed、fixed、regression、aborted、failure、success、unstable和cleanup。这些条件块允许根据管道或阶段的完成状态，在每个条件中执行步骤。条件块按照下面所示的顺序执行。
说明 	选项
必须 	否
参数 	None
允许 	在顶层管道块和每个stage块中

条件
always
不管管道或stage的运行完成状态如何，在post部分运行这些步骤。
changed
如果当前管道或阶段的运行与之前的运行有不同的完成状态，则只运行post中的步骤。
fixed
如果当前管道或阶段的运行成功，且前一次运行失败或不稳定，则只运行post中的步骤。
regression
如果当前管道或阶段的运行状态为失败、不稳定或中止，且前一次运行成功，则只运行post中的步骤。
aborted
只有在当前管道或阶段的运行有“中止”状态时才运行post中的步骤，通常是由于管道被手动中止。这通常用web UI中的灰色表示。
failure
只有在当前管道或阶段的运行有“失败”状态时才运行post中的步骤，通常在web UI中表示为红色。
success
只有当当前管道或阶段的运行具有“成功”状态时，才运行post中的步骤，通常在web UI中以蓝色或绿色表示。
unstable
如果当前管道或阶段的运行状态为“不稳定”状态，通常是由测试失败、代码违规等引起的，那么只在post中运行这些步骤。
cleanup
不管管道或阶段的状态如何，在所有其他的post条件被评估之后，运行这个post条件下的步骤。

例子
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
    post {      ①
        always {          ②
            echo 'I will always say Hello again!'
        }
    }
}
```
按照惯例，post部分应该放在管道的末端。
Post-条件块包含与步骤部分相同的步骤。
stages

包含一个或多个阶段指令的序列，阶段部分是管道所描述的大部分“工作”的位置。至少建议阶段至少包含一个阶段指令，用于连续交付过程的每个离散部分，例如构建、测试和部署。
说明 	选项
必须 	是
参数 	None
允许 	只有一次，在管道内部。

例子
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {      ①
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```
阶段部分通常会遵循代理、选项等指示。
steps

steps部分定义了在给定的stage指令中执行的一系列的一个或多个步骤。
例子
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {    ①
        stage('Example') {
            steps { 
                echo 'Hello World'
            }
        }
    }
}
```

步骤部分必须包含一个或多个步骤。
选项

options指令允许从管道内部配置特定于管道的选项。管道提供了许多这样的选项，比如buildDiscarder，但也可以由插件提供，比如timestamps。
说明 	选项
必须 	否
参数 	None
允许 	只有一次，在管道内部。

## 可用选项
buildDiscarder
为最近的管道运行的特定数量保存artifacts和控制台输出。例如:options {buildder (logRotator(numToKeepStr: '1')})

disableConcurrentBuilds
不允许同时执行管道。可以用于防止同时访问共享资源等。例如:options { disableConcurrentBuilds() }

overrideIndexTriggers
允许重写分支索引触发器的默认处理。如果分支索引触发器在多分支或组织标签中禁用，options { overrideIndexTriggers(true) }将只允许他们从事这项工作。否则, options { overrideIndexTriggers(false) }只会禁用该作业的分支索引触发器。

skipDefaultCheckout
在代理指令中，跳过从源代码控制中检出代码。例如:options { skipDefaultCheckout() }

skipStagesAfterUnstable
跳过阶段一旦构建状态变得不稳定。例如:options { skipStagesAfterUnstable() }

checkoutToSubdirectory
在工作空间的子目录中执行自动源代码控制签出。例如:options { checkoutToSubdirectory('foo') }

newContainerPerStage
与docker或dockerfile顶层agent一起使用。当指定时，每个阶段将在同一个节点上的一个新的容器实例中运行，而不是在同一个容器实例中运行的所有阶段。

timeout
设置管道运行的超时时间，在此之后，Jenkins将中止管道。例如:options { timeout(time: 1, unit: 'HOURS') }

retry
在失败时，重新尝试整个管道的指定次数。例如:options { retry(3) }

timestamps
准备由流水线生成的所有控制台输出，并在该行发出的时间运行。例如：options { timestamps() }
例子
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    options {
        timeout(time: 1, unit: 'HOURS')     ①
    }
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```
 指定一个小时的全局执行超时，之后Jenkins将中止管道运行。
stage选项

stage的选项指令类似于管道根上的options指令。但是，stage层次选项只能包含retry、timeout或timestamps等步骤，或与阶段相关的声明性选项，如skipDefaultCheckout。

在一个阶段中，选项指令中的步骤在进入代理之前被调用，或者在条件出现时进行检查。

可用的Stage选项

skipDefaultCheckout
在代理指令中，跳过从源代码控制中检出代码。例如:options { skipDefaultCheckout() }

timeout
设置此阶段的超时时间，在此之后，Jenkins将中止该阶段。例如:options { timeout(time: 1, unit: 'HOURS') }

retry
在失败时，重试此阶段指定次数。例如:options { retry(3) }

timestamps
在该阶段中生成所有控制台输出，并在其中输出该线路的时间。例如：options { timestamps() }

例子
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example') {
            options {
                timeout(time: 1, unit: 'HOURS') 
            }
            steps {
                echo 'Hello World'
            }
        }
    }
}
```
## parameters(参数)

参数指令提供了一个用户在触发管道时应该提供的参数列表。这些用户指定参数的值可通过params对象提供给管道步骤，请参见示例以了解其具体使用情况。
说明 	选项
必须 	否
参数 	None
允许 	只有一次，在管道内部。

可用的参数

string
字符串类型的参数，例如:parameters { string(name: 'DEPLOY_ENV', defaultValue: 'staging', description: '') }

booleanParam
一个boolean参数, 例如: parameters { booleanParam(name: 'DEBUG_BUILD', defaultValue: true, description: '') }

例子:
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
    }
    stages {
        stage('Example') {
            steps {
                echo "Hello ${params.PERSON}"
            }
        }
    }
}
```
## 触发器(triggers)

triggers指令定义了重新触发管道的自动化方式。对于集成了源(如GitHub或BitBucket)的管道，可能不需要triggers，因为基于web的集成很可能已经存在。当前可用的触发器是cron、pollSCM和upstream。
说明 	选项
必须 	否
参数 	None
允许 	只有一次，在管道内部。

cron
接受一个cron样式的字符串来定义要重新触发管道的常规间隔，例如:triggers { cron('H */4 * * 1-5') }

pollSCM
接受一个cron样式的字符串来定义一个固定的间隔，在这个间隔中，Jenkins应该检查新的源代码更改。如果新的变化存在，管道将被重新触发。例如:triggers { pollSCM('H */4 * * 1-5') }

upstream
接受逗号分隔的工作字符串和阈值。当字符串中的任何作业以最小阈值结束时，管道将被重新触发。例如:triggers { upstream(upstreamProjects: 'job1,job2', threshold: hudson.model.Result.SUCCESS) }

例子
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    triggers {
        cron('H */4 * * 1-5')
    }
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```
## stage

stage指令在stage部分中进行，应该包含一个步骤部分、一个可选的代理部分或其他特定于阶段的指令。实际上，管道所做的所有实际工作都将封装在一个或多个阶段指令中。
说明 	选项
必须 	至少一个
参数 	一个强制的参数，一个用于舞台名称的字符串。
允许 	阶段内的部分
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'
            }
        }
    }
}
```

## input

一个阶段的输入指令允许您使用输入步骤提示输入。在应用了任何选项之后，stage(舞台)将暂停，在进入stages的“代理或评估其条件”之前。如果输入被批准，这个阶段就会继续。作为输入提交的一部分提供的任何参数都将在环境中用于其他阶段。

*配置选项

mesage
必需的。这将在用户提交input时呈现给用户。

id
此input的可选标识符。默认为阶段名称。

ok
input表单上的“ok”按钮的可选文本。

submitter
允许提交此input的用户或外部组名的可选逗号分隔列表。默认允许任何用户。

submitterParameter
一个环境变量的可选名称，如果存在，则用submitter名称设置。

parameters
一个可选的参数列表，以提示提交者提供。有关更多信息，请参见参数。

例子
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example') {
            input {
                message "Should we continue?"
                ok "Yes, we should."
                submitter "alice,bob"
                parameters {
                    string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
                }
            }
            steps {
                echo "Hello, ${PERSON}, nice to meet you."
            }
        }
    }
}
```
## when

when指令允许管道决定是否应该根据给定的条件执行阶段。when指令必须包含至少一个条件时。如果when指令包含多个条件时，所有子条件必须返回true，以便执行。这与子条件在allOf条件下嵌套的情况相同(参见下面的例子)。如果使用了anyOf条件，请注意，一旦找到第一个“true”条件，条件就会跳过其余的测试。

使用嵌套条件可以构建更复杂的条件结构:not、allOf或anyOf。嵌套条件可以嵌套到任意深度。
说明 	选项
必须 	不
参数 	None
允许 	在一个阶段的指令

内置的条件
branch

当正在构建的分支与给定的分支模式匹配时，执行这个阶段，例如:when {branch 'master'}。注意，这只适用于多分支管道。
buildingTag

在构建构建标记时执行阶段。示例:when { buildingTag() }
changelog

如果构建的SCM changelog包含给定的正则表达式模式，则执行该阶段:when { changelog '.*^\\[DEPENDENCY\\] .+$' }
changeset

如果构建的SCM changeset包含一个或多个匹配给定字符串或glob的文件，则执行该阶段。例子:when { changeset "**/*.js" }

默认情况下，路径匹配是不区分大小写的，这可以用caseSensitive参数关闭，例如:when { changeset glob: "ReadMe.*", caseSensitive: true }
changeRequest

如果当前的构建是为了“变更请求”(a.k.a. Pull request on GitHub和Bitbucket，在GitLab上合并请求或Gerrit的变更等)，执行阶段。当没有传递参数时，每个更改请求都会运行阶段，例如:when { changeRequest() }

通过在更改请求中添加带有参数的filter属性，可以只在匹配的更改请求上运行该阶段。可能的属性包括id、target、branch、fork、url、title、author、authorDisplayName和authorEmail。每一个都对应一个CHANGE_*环境变量，例如:when { changeRequest target: 'master' }

可以在属性之后添加可选的参数比较器，以指定匹配的模式的值。EQUALS对于一个简单的字符串比较(默认)，GLOB对于ANT样式路径glob(例如，changeset), 或REGEXP正则表达式匹配。例如:when { changeRequest authorEmail: "[\\w_-.]+@example.com", comparator: 'REGEXP' }
environment

当指定的环境变量设置为给定值时，执行阶段:when { environment name: 'DEPLOY_TO', value: 'production' }
equals

当预期值等于实际值时执行阶段，例如:when { equals expected: 2, actual: currentBuild.number }
expression

当指定的Groovy表达式计算为true时执行阶段，例如:when { expression { return params.DEBUG_BUILD } }注意，当从表达式返回字符串时，必须将它们转换为布尔值或返回null，以使其值为false。简单地返回“0”或“false”仍将计算为“true”。
tag

如果TAG_NAME变量匹配给定的模式，则执行阶段。例子:when { tag "release-*" }。如果提供了一个空模式，那么如果TAG_NAME变量存在(与buildingTag()相同)，则阶段将执行。

not
当嵌套条件为假时执行阶段。必须包含一个条件。例如:when { not { branch 'master' } }

allOf
当所有嵌套条件都为真时，执行阶段。必须包含至少一个条件。例如:when { allOf { branch 'master'; environment name: 'DEPLOY_TO', value: 'production' } }

anyOf
当至少一个嵌套条件为真时，执行阶段。必须包含至少一个条件。例如:when { anyOf { branch 'master'; branch 'staging' } }
评估when进入阶段的代理

默认情况下，如果定义了某个阶段的代理，则在进入该阶段的代理之后将对其进行评估。但是，可以通过在when块中指定beforeAgent选项来更改此选项。如果beforeAgent被设置为true，那么将首先对when条件进行评估，只有在条件计算为true时才会输入代理。

例子
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                branch 'production'
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                branch 'production'
                environment name: 'DEPLOY_TO', value: 'production'
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                allOf {
                    branch 'production'
                    environment name: 'DEPLOY_TO', value: 'production'
                }
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                branch 'production'
                anyOf {
                    environment name: 'DEPLOY_TO', value: 'production'
                    environment name: 'DEPLOY_TO', value: 'staging'
                }
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            when {
                expression { BRANCH_NAME ==~ /(production|staging)/ }
                anyOf {
                    environment name: 'DEPLOY_TO', value: 'production'
                    environment name: 'DEPLOY_TO', value: 'staging'
                }
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent none
    stages {
        stage('Example Build') {
            steps {
                echo 'Hello World'
            }
        }
        stage('Example Deploy') {
            agent {
                label "some-label"
            }
            when {
                beforeAgent true
                branch 'production'
            }
            steps {
                echo 'Deploying'
            }
        }
    }
}
```
## Parallel(并行)

声明式管道中的阶段可以声明它们内部的多个嵌套阶段，它们将并行执行。请注意，一个stage(舞台)必须只有一个steps或parallel的步骤。嵌套阶段本身不能包含进一步的并行阶段，但其他阶段的行为与其他stage(舞台)相同。任何包含并行的stage(舞台)都不能包含代理或工具，因为它们没有相关步骤。

另外，当其中一个进程失败时，您可以强制所有的parallel阶段都被终止，并在包含parallel的stage(舞台)中添加failFast true。

例子
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Non-Parallel Stage') {
            steps {
                echo 'This stage will be executed first.'
            }
        }
        stage('Parallel Stage') {
            when {
                branch 'master'
            }
            failFast true
            parallel {
                stage('Branch A') {
                    agent {
                        label "for-branch-a"
                    }
                    steps {
                        echo "On Branch A"
                    }
                }
                stage('Branch B') {
                    agent {
                        label "for-branch-b"
                    }
                    steps {
                        echo "On Branch B"
                    }
                }
            }
        }
    }
}
```
## Steps

声明式管道可以使用管道步骤引用中记录的所有可用步骤，其中包含一个完整的步骤列表，其中添加了下面列出的步骤，这些步骤只在声明性管道中得到支持。
script

script步骤需要一个脚本化的管道，并在声明式管道中执行。对于大多数用例，在声明式管道中，script步骤应该是不必要的，但是它可以提供一个有用的“逃生出口”。应该将非平凡大小和/或复杂性的脚本块转移到共享库中。

例子
```Jenkinsfile
Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Example') {
            steps {
                echo 'Hello World'

                script {
                    def browsers = ['chrome', 'firefox']
                    for (int i = 0; i < browsers.size(); ++i) {
                        echo "Testing the ${browsers[i]} browser"
                    }
                }
            }
        }
    }
}
```

脚本管道

脚本化的管道，比如声明式管道，是建立在底层管道子系统之上的。不同于声明性的，脚本化的管道实际上是用Groovy构建的通用DSL[2]。Groovy语言提供的大部分功能都可以用于脚本化管道的用户，这意味着它可以是一个非常有表现力和灵活的工具，可以通过它编写持续的交付管道。
Flow Control(流量控制)

脚本化的管道从一个Jenkinsfile的顶部向下串行执行，就像Groovy或其他语言中的大多数传统脚本一样。因此，提供流控制取决于Groovy表达式，例如if/else条件，例如:
```Jenkinsfile
Jenkinsfile (Scripted Pipeline)
node {
    stage('Example') {
        if (env.BRANCH_NAME == 'master') {
            echo 'I only execute on the master branch'
        } else {
            echo 'I execute elsewhere'
        }
    }
}
```

另一种方法是使用Groovy的异常处理支持来管理脚本化的管道流控制。当步骤失败，无论什么原因，他们抛出一个异常。处理错误的行为必须使用Groovy中的try/catch/finally块，例如:

```Jenkinsfile
Jenkinsfile (Scripted Pipeline)
node {
    stage('Example') {
        try {
            sh 'exit 1'
        }
        catch (exc) {
            echo 'Something failed, I should sound the klaxons!'
            throw
        }
    }
}
```



## 结语

正如本章一开始所讨论的，管道最基本的部分是“步骤”。从根本上说，步骤告诉Jenkins应该做什么，并作为声明式和脚本化的管道语法的基本构建块。

脚本化的管道不引入任何特定于其语法的步骤; 管道步骤引用包含管道和插件提供的步骤的完整列表。
