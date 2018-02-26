# learn-groovy

### Declarative pipeline syntax
```java
pipeline {

// agent configuration，总在pipeline开头，或者stage的开头
  agent {
    any
  }
  agent {
    none
  }
  agent {
    label "my-define-label"
  }
  agent {
    dockerfile true
  }
  agent {
    dockerfile {
      dir 'someSubDir'
      filename "someOtherDockerfile"
      label "nodeYouRunning"
      args "-v /tmp:/tmp"
      additionalBuildArgs ' --build-arg foo=bar'
    }
  }
  agent {
    docker {
      image "myImage"
      label "nodeYouRunning"
      args "-v /tmp:/tmp"
    }
  }
  agent {
    node {
      label "myNode"
      customWorkspace "/some/other/path"
    }
  }

  //environment 针对那些生效，取决于environment的位置是在开头，还是某个stage开头
  environment {
    FOO = "bar"
    OTHER = "${FOO}baz"
    SOME_CREDENTIALS = credentials ('some-id')
  }

  // tool 指定工具，工具类型和名字，这需要提前在system config中配置
  tools {
    maven "M3"
    jdk "JDK8"
  }

  // options 配置pipeline本身的一些特定选项
  options {
    //确保显示多少个历史构建
    buildDiscarder (logRotator(numTokeepStr: '10'))
    skipDefaultCheckout()
    timestamps()
    timeout(time: 5, unit: 'MINUTES')
  }

  // triggers 触发某些操作
  triggers {
    cron('@daily')
    pollSCM('H 4/* 0 0 1-5')
  }
  //parameters 设定job构建时的参数，可指定默认值
  parameters {
    booleanParam(defaultValue: true,description: '', name: 'flag')
    string(defaultValue: '',description: '', name: 'SOME_STRING')
  }
  //stages 设定一些执行的阶段和步骤
  stages {
    stage('first stage') {
      environment {
        FOO = "notBar"
      }
      tools {
        maven "M3"
      }
      agent {
        label "some-other-label"
      }
      // 使用一些条件来决定是否指定stage，下面注释的部分为 `非与或` 用法
      when {
        branch "master"
        environment name: "FOO",value: "notBar"
        expression {
          echo "You can run any Pipeline steps in here"
          return "foo" == "bar"
        }
      }
      // when {
      //   anyOf {
      //     branch "master"
      //   }
      // }
      // when {
      //   not {
      //     branch "master"
      //   }
      // }
      // when {
      //   allOf {
      //     branch "master"
      //   }
      // }
      steps {
        echo "i'm doing things, i guess."
        retry(5) {
          echo "Keep trying this if it fails up to 5 times"
        }

        // 在steps中执行一些script pipeline语句
        script {
          if ("sky" == "blue") {
            echo "You can't actually do loops or if statements etc in Declarative unless you're in a script block!"
          }
        }
      }
    }
  }
// post，总在pipeline的末尾或者stage的末尾
  post {
    always {
      echo "do someThing"
    }
    changed {
      echo "do someThing on status different previously"
    }
    faillure {
      echo "do someThing while has a failled status"
    }
    success {
      echo "do someThing while has a success status"
    }
    unstable {
      echo "do someThing while has an unstable status"
    }
  }
}

```

### Scripted pipeline syntax
```java
// build with groovy and is effectively a general purpose DSL
node {
  stage('Example'){
    if (env.BRANCH_NAME=='master') {
      echo 'I only excute on the master branch'
    } else {
      echo 'I execute elsewhere'
    }
  }
  stage('Second Stage') {
    try {
      sh 'exit 1'
    }
    catch (exc) {
      echo "Something failled,I should sound the Klaxons!"
      throw
    }
  }
}
```
