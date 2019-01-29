# Endly introduction

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
4. Check endly version (0.27+)    
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
       
   ```
### Workflow

##### Basic workflow
1. Workflow definition
    [@workflow/helloworld.yaml](workflow/helloworld.yaml)
    ```yaml
    init:
      name: $params.name
    pipeline:
      task1:
        action: print
        message: Hello World $name
        description: print hello
      task2:
        description: test subtasks
        subTask1:
          action: print
          message: subTask 1 $params.key1
        subTask2:
          action: print
          message: subTask 1 $params.name
      task3:
        description: print bye
        action: print
        message: Bye $name
    ```
2. Workflow execution
    * with relative path
    ```bash
    endly -r=workflow/helloworld
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
5. Printing workflow data model
    ```bash
    endly -r=workflow/helloworld -p
    ```
6. Debugging workflow execution
    ```bash
    endly -r=workflow/helloworld -d=true
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
    ```
4. Exec SSH service usage example
    * Service action list: ```endly -s=exec```
    * 'exec:run' contract ```endly -s=exec -a=run```
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


##### Validation    


##### Execution Control

