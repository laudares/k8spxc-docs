# Monitor database with Percona Monitoring and Management (PMM)

In this section you will learn how to monitor Percona XtraDB Cluster with [Percona Monitoring and Management (PMM)](https://www.percona.com/doc/percona-monitoring-and-management/2.x/index.html).

!!! note

    Only PMM 2.x versions are supported by the Operator.

PMM is a client/server application. It includes the [PMM Server](https://www.percona.com/doc/percona-monitoring-and-management/2.x/details/architecture.html#pmm-server) and the number of [PMM Clients](https://www.percona.com/doc/percona-monitoring-and-management/2.x/details/architecture.html#pmm-client) running on each node with the database you wish to monitor.

A PMM Client collects needed metrics and sends gathered data to the PMM Server.
 As a user, you connect to the PMM Server to see database metrics on
a number of dashboards.

PMM Server and PMM Client are installed separately.

## Install PMM Server

You must have PMM server up and running. You can run PMM Server as a *Docker image*, a *virtual appliance*, or on an *AWS instance*.
Please refer to the [official PMM documentation](https://www.percona.com/doc/percona-monitoring-and-management/2.x/setting-up/server/index.html)
for the installation instructions.

## Install PMM Client

To install PMM Client as a side-car container in your Kubernetes-based environment, do the following:
{.power-number}

1. Authorize PMM Client within PMM Server. 

    === "Token-based authorization (recommended)"

        1. [Get the PMM API key from PMM Server :octicons-link-external-16:](https://docs.percona.com/percona-monitoring-and-management/details/api.html#api-keys-and-authentication). The API key must have the role "Admin". You need this key to authorize PMM Client within PMM Server. 

            === ":material-view-dashboard-variant: From PMM UI"          

                [Generate the PMM API key :octicons-link-external-16:](https://docs.percona.com/percona-monitoring-and-management/details/api.html#api-keys-and-authentication){.md-button} 

            === ":material-console: From command line"        

                You can query your PMM Server installation for the API
                Key using `curl` and `jq` utilities.         

                Retrieve the user credentials from the PMM Server:        

                ```.bash
                PMM_AUTH=$(echo -n admin:$(kubectl get secret pmm-secret -o jsonpath='{.data.PMM_ADMIN_PASSWORD}' | base64 --decode) | base64)
                ```        

                Replace the `$PMM_SERVER_IP` placeholder with your real PMM Server hostname in the following command:
                
                ```.bash 
                export API_KEY=$(curl --insecure -X POST -H "Authorization: Basic $PMM_AUTH" -H 'Content-Type: application/json'  -d '{"name":"operator", "role": "Admin"}' "https://$PMM_SERVER_IP/graph/api/auth/keys" | jq .key)
                ```        

                To get the API_KEY value, run:        

                ```.bash
                echo $API_KEY
                ```        

                The output shows the base64 encoded API key.         

            <i warning>:material-alert: Warning:</i> The API key is not rotated automatically.

        2. Edit the [deploy/secrets.yaml](https://github.com/percona/percona-xtradb-cluster-operator/blob/main/deploy/secrets.yaml) secrets file and specify the PMM API key for the `pmmserverkey` option.
        3. Apply the configuration for the changes to take effect.

            ``` {.bash data-prompt="$" }
            $ kubectl apply -f deploy/secrets.yaml -n <namespace>
            ```
    
    === "Password-based authorization (deprecated since the Operator 1.11.0)"

        1. Check that  the `serverUser` key in the
            [deploy/cr.yaml](https://github.com/percona/percona-xtradb-cluster-operator/blob/main/deploy/cr.yaml)
            file contains your PMM Server user name (`admin` by default), and
            make sure the `pmmserver` key in the
            [deploy/secrets.yaml](https://github.com/percona/percona-xtradb-cluster-operator/blob/main/deploy/secrets.yaml)
            secrets file contains the password specified for the PMM Server during
            its installation

        2. Apply the configuration for the changes to take effect.

            ``` {.bash data-prompt="$"}
            $ kubectl apply -f deploy/secrets.yaml -n <namespace>
            ```

2. Update the `pmm` section in the [deploy/cr.yaml](https://github.com/percona/percona-server-mongodb-operator/blob/main/deploy/cr.yaml) file:

    * Set `pmm.enabled`=`true`.
    * Specify your PMM Server hostname / an IP address for the `pmm.serverHost` option. The PMM Server IP address should be resolvable and reachable from within your cluster.

     ```yaml
       pmm:
         enabled: true
         image: percona/pmm-client:{{pmm2recommended}}
         serverHost: monitoring-service
     ``` 
3. Apply the changes:

    ``` {.bash data-prompt="$"}
    $ kubectl apply -f deploy/cr.yaml -n <namespace>
    ```

4. Check that corresponding Pods are not in a cycle of stopping and restarting.
    This cycle occurs if there are errors on the previous steps:

    ``` {.bash data-prompt="$" }
    $ kubectl get pods -n <namespace>
    $ kubectl logs  <cluster-name>-pxc-0 -c pmm-client -n <namespace>
    ```

## Check the metrics

Let's see how the collected data is visualized in PMM.
{.power-number}

Now you can access PMM via *https* in a web browser, with the login/password
authentication, and the browser is configured to show Percona XtraDB Cluster
metrics.
