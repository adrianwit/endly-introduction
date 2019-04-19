# Endly introduction

[Watch endly introduction](https://youtu.be/eXVwltuXhXM)


### Prerequisites

1. [Docker service](https://www.docker.com/get-started)


### Getting Started

1. Create test and credentials directories 
    ```bash
    mkdir -p ~/e2e
    mkdir -p ~/.secret
    ```
2. Start endly docker image
    ```bash
    docker run --name endly -v /var/run/docker.sock:/var/run/docker.sock -v ~/e2e:/e2e -v ~/.secret/:/root/.secret/ -p 7722:22  -d endly/endly:latest-ubuntu16.04  
    ```
3. Logging to endly container
    ```bash
    ssh root@127.0.0.1 -p 7722 ## password is dev    
    ```
4. Check endly version    
   ```bash
    endly -v    
   ```
5. Create dev container credentials (use user:root, password:dev)
   ```bash
   endly -c=dev    
   ```
6. Clone this repo
   ```bash
   cd /e2e
   git clone https://github.com/adrianwit/endly-introduction.git
   ```
### Workflow
1. Hello world

     * ```endly -r=workflow/hello```
    [@workflow/hello.yaml](workflow/hello.yaml)
   ```yaml
    init:
      target:
        URL: ssh://127.0.0.1
        credentials: dev
    pipeline:
      task1:
        action: workflow.print
        message: Hello World
      task2:
        action: exec:run
        target: $target
        commands:
          - echo 'Hello World'
    ```
   
##### Basic workflow
1. Workflow definition
    [@workflow/helloworld.yaml](workflow/helloworld.yaml)
    ```yaml
    init:
      name: $params.name
    pipeline:
      task1:
        action: workflow.print
        message: Hello World $name
        description: print hello
      task2:
        description: test subtasks
        subTask1:
          action: print
          message: subTask 1 $params.key1
        subTask2:
          action: print
          message: subTask 2 $params.name
      task3:
        description: print bye
        action: print
        message: Bye $name
    ```
2. Workflow execution
    * with relative path and parameters
    ```bash
    endly -r=workflow/helloworld name=Endly key1=test
    ```
    * with URL
    ```bash
    endly -r=https://github.com/adrianwit/endly-introduction/blob/master/workflow/helloworld.yaml
    ```
3. Workflow task info
    ```bash
    endly -r=workflow/helloworld -t='?'
    ```
4. Workflow task selection
    ```bash
    endly -r=workflow/helloworld -t='task1,task3'
    ```

##### Services

1. Endly supported services:
    ```bash
    endly -s='*'
    ```
2. Action attribute.
   * defines what service method is used for an workflow action    
   * **_action_ attribute format**:  ```[serivce]:action```
   * default endly service is workflow
3. Service Contract
   * specifies request type: optional and required action node attributes for an action
   * specifies action response type  
    ```bash
    endly -s=workflow -a=print
    #or
    endly -s=workflow:print
 
    ```
4. Running adhoc service action from CLI
    * With request attribute supplied as cli arguments
        - ```endly -run="print" message='hello'``` 
        - ```endly -run="validator:assert" actual=1 expect=3```
    * With request delegated to JSON or YAML file
        - ```endly -run='validator:assert' request='@services/assert.yaml'```
        [@service/assert.yaml](service/assert.yaml) 
        ```yaml
        actual:
            - 1
            - 2
            - 3
        expect:
            - 1
            - 2
            - 3    
        ```
        
4. Exec SSH service usage example
    * Service action list: ```endly -s=exec```
    * 'exec:run' contract ```endly -s=exec -a=run```
    * ```endly -r=services/info.yaml```
    * [@services/info.yaml](services/info.yaml)
    ```yaml
    init:
      target:
        url:  ssh://127.0.0.1/
        credentials: dev
    
    defaults:
      target: $target
    
    pipeline:
      init:
        action: exec:run
        commands:
          - whoami
          - echo 'hello ${cmd[0].stdout}'
      hostInfo:
        action: exec:run
        commands:
          - cat /etc/hosts
        extract:
          - key: aliases
            regExpr: (?sm)127.0.0.1[\s\t]+(.+)
      debug:
        action: print
        message: $AsJSON($init) $AsJSON($hostInfo)
      summary:
        action: print
        message: Extracted 127.0.0.1 aliases $hostInfo.aliases
    ```
5. Request delegation
   * [@services/delegation.yaml](services/delegation.yaml) workflow
   ```yaml
   init:
     target:
       url:  ssh://127.0.0.1/
       credentials: dev
   defaults:
     target: $target
   pipeline:
     task1:
       action: storage:copy
       request: @copy
     task2:
       action: exec:run
       commands:
         - ls -al /tmp/README.md
   ```
   * [@services/copy.yaml](services/copy.yaml) request
   ```yaml
   source:
     URL: https://raw.githubusercontent.com/adrianwit/endly-introduction/master/
   dest:
     URL: $target
   assets:
     'README.md': /tmp/README.md
   ```
   * execution: ```endly -r=services/delegation```      
 
 
 
 
#####  Variables
1. Variables control workflow execution and data substitution
2. Variable are defined in init section at action, task or workflow node
3. Variable reference expression start with '$' sign
   * any reference to variable is expanded/evaluated if variable is defined
   * nesting variable uses {} 
   * variable expression supports basic arithmetic expression
4. Required variables usage
   * [@variables/required.yaml](variables/required.yaml)
    ```yaml
    init:
      '!to': $params.to
      greeting: $params.greeting
      message: "$greeting:/$/? 'Hello $to' : '$greeting $to'"
    pipeline:
        welome:
         action: print
         message: $message
    ``` 
    - ```endly -r=variables/required.yaml```
    - ```endly -r=variables/required.yaml to=Endly```
    - ```endly -r=variables/required.yaml to=Endly greeting=Greetings```

5. User defined function
    * listing loaded UDF ```endly -j ```
     
    * [@variables/udf.yaml](variables/udf.yaml) example 
   ```yaml
    init:
      fiveDaysAgo: $FormatTime('5daysAgoInUTC','yyyyMMdd')
      10DaysAhead: $FormatTime('5daysAheadInUTC','yyyy-MM-dd hh:mm:ss')
      encodedMsg: $Base64Encode("This is test")
      ID: $uuid.next
    pipeline:
      task1:
        action: print
        message: $ID $fiveDaysAgo $10DaysAhead $encodedMsg
    ```
    * ```endly -r=variables/date_setup.yaml```
        
6. Array/Collection type variable usage
   * [@variables/collection.yaml](variables/collection.yaml)
   ```yaml
    init:
      i: 2
      array:
        - id: 1
          name: name 1
        - id: 2
          name: name 2
        - id: 3
          name: name 3
    
    pipeline:
      shift:
        action: print
        init:
          item: ${<-array}
        message: $AsJSON($item)
    
      push:
        action: nop
        init:
          - name: myItem
            value:
              id: 10,
              name: name 10
    
          - '->array': $myItem
      lastNameInfo:
        action: print
        message: ${array[$i].name}
      summary:
        action: print
        message: $AsJSON($array)
   ```
7. Basic arithmetic
    * [@variables/arithmetic.yaml](variables/arithmetic.yaml) 
    ```yaml
    init:
      i: 1
      j: 10
      k: 100
    pipeline:
      task1:
        action: print
        message: ${(k/j)+i}
    ```    
8. Action response variable
    * After each action execution response can be access via $ACTION_NAME
    * endly -r=variables/inspect
    * [@variables/inspect.yaml](variables/inspect.yaml) 
    ```yaml
    pipeline:
      task1:
        inspect:
          action: docker:inspect
          '@name': endly
          logging: false
        ipInfo:
          action: print
          message: ${inspect.Info[0].NetworkSettings.IPAddress}
        networkInfo:
          action: print
          message: $AsJSON(${inspect.Info[0].NetworkSettings.Networks})
      task2:
        inspect:
          action: docker:inspect
          '@name': endly
          ':name': endlyInfo
          logging: false
        ipInfo:
          action: print
          message: ${endlyInfo.Info[0].NetworkSettings.IPAddress}
        networkInfo:
          action: print
          message: $AsJSON(${endlyInfo.Info[0].NetworkSettings.Networks})

    ```   
    **Note** that if name is both part of docker:inspect contract (endly -s='docker:inspect'), and 
    workflow action node model (action has name attribute), '@' prefix is used for service contract, 
    and ':' for workflow model respectively. 
    In task1 name of action is inherited from node name ('inspect'),
    in task2 name is specified with ':' prefix.
    


#####  Workflow Actions Template
1. Action template repeats N time group of flattened actions defined under template node:
2. [@template/basic.yaml](template/basic.yaml) example
    ```yaml
    pipeline:
      init:
        action: print
        message: init ...
      tasksX:
        tag: $pathMatch
        range: 1..002
        description: '@desc'
        subPath: basic/${index}_*
        template:
          action1:
            action: print
            message: "action 1 - idx: $index, tagId: $tagId, path: $path"
          action2:
            action: print
            request: '@print'
      done:
        action: print
        message: done.
    ```
    * template node
    * range
    * template variables: $index, $tagId, $path, $subPath
    * request location auto-discovery
    * description discovery
3. Tags selection    
    * ```endly -r=template/basic.yaml -i=basic_caseYY``` 
4. Data loading
    * ```endly -r=template/stress_test``` 
    * [@template/stress_test.yaml](template/stress_test.yaml)
    ```yaml
    pipeline:
      test:
        tag: LoadTest
        data:
          ${tagId}.[]Requests: '@data/Request*.json'
        range: '1..002'
        subPath: stess_test/${index}_*
        template:
          init:
            action: nop
            comments: set endpoint port and address
            init:
              idx: $AsInt($index)
              threadCont: ${idx * 4}
              testEndpointPort: 803$AsInt($index)
              testEndpoint: 127.0.0.1:${testEndpointPort}
    
          endpointUp:
            action: http/endpoint:listen
            comments: start test endpoint 127.0.0.1:${testEndpointPort}
            port: $testEndpointPort
            rotate: true
            baseDirectory: ${path}/endpoint/
    
          info:
            action: print
            message: starting load testing $testEndpont
    
          load:
            action: 'http/runner:load'
            comments: run load test (40k * 5)
            assertMod: 40
            threadCount: $threadCont
            '@repeat': 40000
            requests: ${data.${tagId}.Requests}
    
          load-info:
            action: print
            message: 'QPS: $load.QPS: Response: min: $load.MinResponseTimeInMs ms, avg: $load.AvgResponseTimeInMs ms max: $load.MaxResponseTimeInMs ms'
    ```    
    * data node

##### Validation    

1. Basic validation example        
    ```bash
    endly -r=validation/test
    ```
    [@validation/test.yaml](validation/test.yaml)
    ```yaml
    pipeline:
      init:
        action: workflow:print
        message: init test ...
      test:
        workflow: assert
        actual:
          key1: value1
          key2: value20
          key3: value30
        expected:
          key1: value1
          key2: value2
          key3: value3
          key4: value4     
    ```
    
2. Simple validation
    ```bash
    # to get validator:assert contract:
    endly -s=validator:assert
    #run assets    
    endly -run='validator:assert' actual A expect B
    #or
    endly validator:assert actual A expect B
 
    ``` 
3.  Data structure validation
    ```bash
    endly -run='validator:assert' actual '@validation/actual1.json' expected '@validation/expect1.json'
    ``` 
4. Data structure validation with @indexBy@ directive
    ```bash
    endly -run='validator:assert' actual '@validation/actual2.json' expected '@validation/expect2.json'
    ```
5. Data structure alternative transformation validation (switch/case)
    ```bash
    endly -run='validator:assert'  actual '@validation/actual3.json' expected '@validation/expect3.json'
    ```
6. Partial validation with @assertPath@ directive 
    ```bash
    endly -run='validator:assert'  actual '@validation/actual4.json' expected '@validation/expect4.json'
    ```



See more about [validation expression](https://github.com/viant/assertly#validation)
    

_Validation expressions:_
- [Assertly](https://github.com/viant/assertly#validation)

##### Execution Control

1. Conditional action execution
   * [@control/when.yaml](control/when.yaml)
    ```yaml
    pipeline:
      task1:
        action1:
          when: $p = b1
          action: print
          message: hello from action 1
        action2:
          when: $p = b2
          action: print
          message: hello from action 2
        action3:
          when: $HasResource('control/hello.txt')
          action: print
          message: $Cat('control/hello.txt')
    ```
    * ``` endly -r control/when```
    * ``` endly -r control/when  p=b1```
2. Parallel execution,
   * [@control/parallel.yaml](control/parallel.yaml)
    ```yaml
    pipeline:
      task1:
        multiAction: true
        action1:
          action: print
          message: hello from action 1
          sleepTimeMs: 3000
          async: true
        action2:
          action: print
          message: hello from action 2
          sleepTimeMs: 3000
        action3:
          action: print
          message: hello from action 3
          sleepTimeMs: 3000
          async: true
    ```
    * ```endly -r=control/parallel```
    * works on task level with flatten action:multiAction: true
3. Defer execution
   * [@control/defer.yaml](control/defer.yaml)
    ```yaml
    pipeline:
      task1:
        action1:
          action: print
          message: hello action 1
        action2:
          when: $failNow=1
          action: fail
          message: execption in action 2
      defer:
        action: print
        message: allway run
    
    ```
    * ```endly -r=control/defer.yaml ```
    * ```endly -r=control/defer.yaml failNow=1```
4. Handling error
    * [@control/error.yaml](control/error.yaml)
    ```yaml
    pipeline:
      task1:
        action1:
          action: print
          message: hello action 1
        action2:
          when: $p = failNow
          action: fail
          message: execption in action 2
        action3:
          action: print
          message: hello action 3
          
      task2:
        action1:
          action: print
          message: hello task 2 action 1
    
      catch:
        action: print
        message: caught $error.Error
    
    ``` 
    * ```endly -r=control/error```
    * ```endly -r=control/error p=failNow```
5. Loops
   * [@control/loop.yaml](control/loop.yaml)
    ```yaml
    init:
      self.i: 1
    pipeline:
      loop:
        info:
          action: print
          message: pushing message ${self.i}
    
    #    trigger:
    #      action: msg:push
    #      dest:
    #        URL: mye2eQueue
    #        type: queue
    #        vendor: aws
    #      messages:
    #        - data: 'Test: this is my ${self.i} message'
    #          attributes:
    #            id: $self.i
    
        inc:
          action: nop
          logging: false
          init:
            _: ${self.i++}
        until:
          action: goto
          logging: false
          task: loop
          when: $self.i <= 10
        done:
          action: print
          comments: done
    
    ```
    * ```endly -r=control/loop```


##### Debugging & Troubleshooting
1. Enabling logging
    ```endly -r=workflow/helloworld -d```
2. Inspecting workflow domain model
    ```endly -r=workflow/helloworld -f=yaml -p```    
3. Diagnose endly execution with [gops](https://github.com/google/gops)
    * ```gops stack ENDLY_PID```


