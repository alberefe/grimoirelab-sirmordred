## Troubleshooting

Following is a list of common problems encountered while setting up GrimoireLab

---
**NOTE**

In order to see the logs, run ```docker-compose up``` without the ```-d``` or ```--detach``` option while
starting/(re)creating/building/attaching containers for a service.

---

#### Low Virtual Memory

* Indications:gi
  Cannot open ```https://localhost:9200/``` in browser. Shows ```Secure connection Failed```,
  ```PR_END_OF_FILE_ERROR```, ```SSL_ERROR_SYSCALL in connection to localhost:9200``` messages.

* Diagnosis:
    Check for the following log in the output of ```docker-compose up```
   ```
   elasticsearch_1  | ERROR: [1] bootstrap checks failed
   elasticsearch_1  | [1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
   ````
* Solution:
    Increase the kernel ```max_map_count``` parameter of vm. Execute the following command
    ```sudo sysctl -w vm.max_map_count=262144```
    Now stop the container services and re-run ```docker-compose up```.
    Note that this is valid only for current session. To set this value permanently, update the ```vm.max_map_count``` setting
    in /etc/sysctl.conf. To verify after rebooting, run sysctl vm.max_map_count.

#### Processes have conflicts with SearchGuard

* Indications:
    - Cannot open ```localhost:9200``` in browser, shows ```Secure connection Failed```
    - ```curl -XGET localhost:9200 -k``` gives
    ```curl: (52) Empty reply from server```

* Diagnosis:
    Check for the following log in the output of ```docker-compose up```
    ```
    elasticsearch_1  | [2020-03-12T13:05:34,959][WARN ][c.f.s.h.SearchGuardHttpServerTransport] [Xrb6LcS] Someone (/172.18.0.1:59838) speaks http plaintext instead of ssl, will close the channel
    ```
    Check for conflicting processes by running ```sudo lsof -i:58888``` (e.g.  58888 is the port number)

* Solution:
    1. Try to close the conflicting processes:
        You can do this easily with fuser (```sudo apt-get install fuser```), 
        run ```fuser -k 58888/tcp``` (e.g. 58888 is the port number).
        Re-run ```docker-compose up``` and check if ```localhost:9200``` shows up.
    2. Use a docker-compose without SearchGuard:
        Use the docker-compose below, this doesn't include SearchGuard.
        <https://github.com/chaoss/grimoirelab-sirmordred#docker-compose-without-searchguard>.
        Note: With this docker-compose, access to the Kibiter and ElasticSearch don't require credentials.
        Re-run ```docker-compose up``` and check if ```localhost:9200``` shows up.

#### Permission Denied 

* Indications:
  Can't create indices in Kibana. Nothing happens after clicking create index.

* Diagnosis:
  Check for the following log in the output of ```docker-compose up```
  ```
  elasticsearch_1 |[INFO ][c.f.s.c.PrivilegesEvaluator] No index-level perm match for User [name=readall, roles=[readall], requestedTenant=null] [IndexType [index=.kibana, type=doc]] [Action [[indices:data/write/index]]] [RolesChecked [sg_own_index, sg_readall]]

  elasticsearch_1 | [c.f.s.c.PrivilegesEvaluator] No permissions for {sg_own_index=[IndexType [index=.kibana, type=doc]], sg_readall=[IndexType [index=.kibana, type=doc]]}

  kibiter_1 | {"type":"response","@timestamp":CURRENT_TIME,"tags":[],"pid":1,"method":"post","statusCode":403,"req":{"url":"/api/saved_objects/index-pattern?overwrite=false","method":"post","headers":{"host":"localhost:5601","user-agent":YOUR_USER_AGENT,"accept":"application/json, text/plain, /","accept-language":"en-US,en;q=0.5","accept-encoding":"gzip, deflate","referer":"http://localhost:5601/app/kibana","content-type":"application/json;charset=utf-8","kbn-version":"6.1.4-1","content-length":"59","connection":"keep-alive"},"remoteAddress":YOUR_IP,"userAgent":YOUR_IP,"referer":"http://localhost:5601/app/kibana"},"res":{"statusCode":403,"responseTime":25,"contentLength":9},"message":"POST /api/saved_objects/index-pattern?overwrite=false 403 25ms - 9.0B"} 
  ```
  or any type of 403 error.
  
* Solution:
  This message generally appears when you try to create an index pattern but you are not logged in Kibana.
  Try logging in to Kibana (the login button is on the bottom left corner).
  The credentials used for login should be username: `admin` and password: `admin`.

#### Empty Index

* Indications and Diagnosis:
  Check for the following error after executing [Micro Mordred](https://github.com/chaoss/grimoirelab-sirmordred/tree/master/utils/micro.py)
  using ```micro.py --raw --enrich --panels --cfg ./setup.cfg --backends git```(Here, using git as backend)
  ```
  [git] Problem executing study enrich_areas_of_code:git, RequestError(400, 'search_phase_execution_exception', 'No mapping 
  found for [metadata__timestamp] in order to sort on')
  ```
* Solution:
  This error appears when the index is empty (here, ```git-aoc_chaoss_enriched``` index is empty). An index can be empty when 
  the local clone of the repository being analyzed is in sync with the upstream repo, so there will be no new commits to 
  ingest to grimoirelab.
  
  There are 2 methods to solve this problem:
 
  Method 1: Disable the param [latest-items](https://github.com/chaoss/grimoirelab-sirmordred/blob/master/utils/setup.cfg#L78) by setting it to false.
  
  Method 2: Delete the local clone of the repo (which is stored in ```~/.perceval/repositories```).
 
  Some extra details to better understand this behavior:
  
  The Git backend of perceval creates a clone of the repository (which is stored in ```~/.perceval/repositories```) and keeps the local
  copy in sync with the upstream one. This clone is then used to ingest the commits data to grimoirelab.
  Grimoirelab periodically collects data from different data sources (in this specific case, a git repository) in an incremental way. 
  A typical execution of grimoirelab for a git repository consists of ingesting only the new commits to the platform. These 
  commits are obtained by comparing the local copy with the upstream one, thus if the two repos are synchronized, then no 
  commits are returned and hence Index will be empty. In the case where all commits need to be extracted even if there is already a
  local clone, latest-items param should be disabled. Another option is to delete the local clone (which is stored at ```~/.perceval/repositories```),
  and by doing so the platform will clone the repo again and extract all commits. 
 
## Low File Descriptors

* Indications:
    - Cannot open ```localhost:9200``` in browser, shows ```Secure connection Failed```
    - ```curl -XGET localhost:9200 -k``` gives
    ```curl: (7) Failed to connect to localhost port 9200: Connection refused```
    
* Diagnosis:
  Check for the following log in the output of ```docker-compose up```
  ```
    elasticsearch_1  | ERROR: [1] bootstrap checks failed
    elasticsearch_1  | [1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
  ```

* Solution:
  1. Increase the maximum File Descriptors (FD) enforced:
  
        You can do this by running the below command.
        ```
        sysctl -w fs.file-max=65536
        ```
        
        To set this value permanently, update `/etc/security/limits.conf` content to below.
        To verify after rebooting, run `sysctl fs.file-max`.
        ```
        elasticsearch   soft    nofile          65536
        elasticsearch   hard    nofile          65536
        elasticsearch   memlock unlimited
        ```
   
  1. Override `ulimit` parameters in the ElasticSearch docker configuration:
  
        Add the below lines to ElasticSearch service in 
        your compose file to override the default configurations of docker.
        ```
        ulimits:
        nofile:
          soft: 65536
          hard: 65536
        ```