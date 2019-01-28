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
        description: print hello
        message: Hello World $name
      task2:
        subTask1:
          action: print
          message: subTask 1 $params.key1
        subTask2:
          action: print
          message: subTask 1 $params.name
      task3:
        action: print
        description: print bye
        message: Bye $name
    ```
2. Workflow execution
    ```bash
    endly -r=workflow/helloworld
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
    
 
#####  Variables
1. Variables control workflow execution and data substitution
2. Variable are defined in init section at action, task or workflow node
3. Variable reference expression start with '$' sign
   * any reference to variable is expanded/evaluated if variable is defined
   * nesting variable uses {} 
   * variable expression supports basic arithmetic expression
4. User defined function
    ```bash
    endly -j
    ```    




#####  Workflow Actions Template

##### Execution Control
