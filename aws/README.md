# AWS

The following instructions will guide you in order to aprovision a Kubernetes cluster in AWS.

**Note:** This guide only covers MacOS setups.

## Prerequisites

1. Create an [Amazon Web Services](https://aws.amazon.com/) account.

2. Create an IAM user from the AWS dashboard with the following permissions:

    ```
    AmazonEC2FullAccess
    AmazonRoute53FullAccess
    AmazonS3FullAccess
    IAMFullAccess
    AmazonVPCFullAccess
    ```

    **Note:** Write down both the access key id and the secret access key.

3. Install the [AWS CLI](https://aws.amazon.com/cli/):

    ```
    $ brew install awscli
    ```

4. Configure the AWS CLI to use the recently created user

    ```
    $ aws configure # This will prompt the access and secret key, and the aws region
    ```

5. Install [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/).

    ```
    $ brew install kubernetes-cli
    ```

6. Install [KOPS](https://github.com/kubernetes/kops):

    ```
    $ brew install kops
    ```

7. Export the following variables (which will hold both the access key id and the secret access key).

    ```
    $ export AWS_ACCESS_KEY_ID=<access-secret-key>
    $ export AWS_SECRET_ACCESS_KEY=<secret-access-key>
    ```

## Some optional steps

### Domain name

**This is optional only if you already own a hosted zone**

Create a Hosted Zone in [Route53](https://aws.amazon.com/route53/) **if you don't have one yet**:

```
$ aws route53 create-hosted-zone \
    --name <hosted-zone-name>
    --caller-reference <caller-reference>   # A unique value (for example, the current timestamp)
```

#### Example output:

    ```json
    {
        "Location": "https://route53.amazonaws.com/2013-04-01/hostedzone/<hosted-zone-id>",
        "HostedZone": {
            "Id": "/hostedzone/<hosted-zone-id>",
            "Name": "<hosted-zone-name>",
            "CallerReference": "<caller-reference>",
            "Config": {
                "PrivateZone": false
            },
            "ResourceRecordSetCount": 2
        },
        "DelegationSet": {
            "NameServers": [
                "<name-server-1>",
                "<name-server-2>",
                "<name-server-3>",
                "<name-server-4>"
            ]
        }
        "ChangeInfo": ...,
    }

    ```

**Note1:** Take note of the returned id.

**Note2:** Take note of the returned name servers as the previous command would have only created the hosted zone in AWS. You must also delegate the domain to your new hosted zone (which means that you also own the said domain).


### SSL/TLS Certificate

1. Request an SSL/TLS certificate:

    ```
    $ aws acm request-certificate \
        --domain-name <hosted-zone-name>
        --subject-alternative-names *.<hosted-zone-name> [<other-relevant-names>]
        --validation-method dns
    ```

    The returned output will contain the certificate ARN:

    ```json
    {
        "CertificateArn": "<certificate-arn>"
    }
    ```

    **Note:** It might be a good idea to include all known names as the asterisk will only cover first level subdomains. Here you should include your future cluster's name if the certificate's domain name is not already the cluster's name.


2. Retrieve the certificate metadata:

    ```
    $ aws acm describe-certificate \
        --certificate-arn <certificate-arn>
    ```    

    The output will be something like this:


    ```json
    {
        "Certificate": {
            "CertificateArn": "<certificate-arn>",
            "DomainName": "<hosted-zone-name>",
            "SubjectAlternativeNames": [
                "<hosted-zone-name>",
                "*.<hosted-zone-name>"
            ],
            "DomainValidationOptions": [
                {
                    "DomainName": "<hosted-zone-name>",
                    "ValidationDomain": "<hosted-zone-name>",
                    "ValidationStatus": "PENDING_VALIDATION",
                    "ResourceRecord": {
                        "Name": "<validation-key-1>.<hosted-zone-name>.",
                        "Type": "CNAME",
                        "Value": "<validation-value-1>"
                    },
                    "ValidationMethod": "DNS"
                },
                {
                    "DomainName": "*.<hosted-zone-name>",
                    "ValidationDomain": "*.<hosted-zone-name>",
                    "ValidationStatus": "PENDING_VALIDATION",
                    "ResourceRecord": {
                        "Name": "<validation-key-2>.<hosted-zone-name>.",
                        "Type": "CNAME",
                        "Value": "<validation-value-2>"
                    },
                    "ValidationMethod": "DNS"
                }
            ],
            "Subject": "CN=<hosted-zone-name>",
            "Issuer": "Amazon",
            "CreatedAt": "<timestamp>",
            "Status": "PENDING_VALIDATION",
            "KeyAlgorithm": "RSA-2048",
            "SignatureAlgorithm": "SHA256WITHRSA",
            "InUseBy": [],
            "Type": "AMAZON_ISSUED",
            "KeyUsages": [],
            "ExtendedKeyUsages": [],
            "RenewalEligibility": "INELIGIBLE",
            "Options": {
                "CertificateTransparencyLoggingPreference": "ENABLED"
            }
        }
    }
    ```

    Take note of the `ResourceRecord` object inside the `DomainValidationOptions` object, as it is needed to validate the domains for which the certificate was issued.

    **Note:** Your configured user must have permissions to request certificates.


3. Add the validation resource records to your Hosted Zone:

    ```
    $ aws route53 change-resource-record-sets \
        --hosted-zone-id <hosted-zone-id> \
        --change-batch \
            '{
                "Changes": [
                    {
                        "Action": "CREATE",
                        "ResourceRecordSet": {
                            "Name": "<validation-key-1>.<hosted-zone-name>",
                            "Type": "CNAME",
                            "TTL": <ttl>,
                            "ResourceRecords": [ { "Value": "<validation-value-1>" } ]
                        }
                    },
                    {
                        "Action": "CREATE",
                        "ResourceRecordSet": {
                            "Name": "<validation-key-2>.<hosted-zone-name>",
                            "Type": "CNAME",
                            "TTL": <ttl>,
                            "ResourceRecords": [ { "Value": "<validation-value-2>" } ]
                        }
                    }
                ]
            }'

    ```

    **Note that the `validation-key-1` might be the same as the `validation-key-2`. In such case only include it once**.

    The output will be something like this:

    ```json
    {
        "ChangeInfo": {
            "Id": "/change/<change-id>",
            "Status": "PENDING",
            "SubmittedAt": "<timestamp>"
        }
    }    
    ```

    To check if the change was applied, execute the following command:

    ```
    $ aws route53 get-change --id <change-id>
    ```

    The returned output will be something like this:

    ```json
    {
        "ChangeInfo": {
            "Id": "/change/<change-id>",
            "Status": "INSYNC",
            "SubmittedAt": "<timestamp>"
        }
    }
    ```

    If the status is not `INSYNC` then you will have to wait.

4. You will have to wait some time. After the DNS change is propagated, it can take from a couple of minutes up to several hours for ACM to validate the domain. To check if the certificate is validated retrieve its metadata (as you did in step 2).




## Cluster provisioning


1. Create a key pair:

    ```
    $ aws ec2 create-key-pair \
        --key-name <key-name> \
        --region <aws-region>
    ```

    Take note of the returned key. Save it into a `.pem` file (let's call it `<key-pem-file>`.

    **Note:** You can skip this step if you want to use an existing pair. You must know its name, and have access to the `.pem` file.


2. Create an [S3](https://aws.amazon.com/s3/) bucket:

    ```
    $ aws s3api create-bucket \
        --bucket <bucket-name> \
        --region <aws-region> \
        # Add the following flag only for regions other than us-east-1
        --create-bucket-configuration LocationConstraint=<aws-region>
    
    # Add versioning for the bucket (STRONGLY recommended)
    $ aws s3api put-bucket-versioning \
        --bucket <bucket-name> \
        --versioning-configuration Status=Enabled
    ```

    **Note:** This will hold th cluster's state, so versioning is **strongly recommended** to allow easy rollbacks.


3. Create the Kubernetes cluster:

    ```
    $ kops create cluster \
        --name <cluster-name> \                     # This should be the fqdn of the cluster
        --cloud aws \
        --zones <aws-zones> \                       # Make sure they belong to <aws-region>
        --master-zones <aws-zones> \                # Make sure they belong to <aws-region>
        --node-size <worker-instance-type> \
        --master-size <master-instance-type> \
        --node-count <amount-of-workers> \
        --dns-zone <hosted-zone-name> \
        --topology private \
        --networking <calico|weave|kopeio-vxlan> \
        --api-ssl-certificate <certificate-arn> \
        --bastion
        --state s3://<bucket-name>
        
    ```

    This will not deploy any infrastructure. It will only create the cluster state, which will be stored in your S3 bucket.

    **Note:** Ignore the message about the SSH public key.


4. Edit the created configuration in order to add the created key in step 1:

    ```
    $ kops edit cluster \
        --name <cluster-name> \
        --state s3://<bucket-name>
    ```

    This will open your editor with the created cluster state. Once saved and closed, the state will be uploaded to the S3 bucket. You can change your editor by setting the `EDITOR` environment variable:

    ```
    $ export EDITOR=<editor-command>
    ```

    For example, if you want to use sublime, run the following command:

    ```
    $ export EDITOR="subl -n -w"
    ```

    Once the editor is opened, add the key pair name:

    ```yaml
    # ...
    spec:
      sshKeyName: <key-name>
    # ...
    ```

    Save and close the editor.


5. Build the cluster:

    ```
    $ kops edit cluster \
        --name <cluster-name> \
        --state s3://<bucket-name> \
        --yes
    ```

    This will set up all the infrastructure needed to create the cluster in AWS. 

    **From now on charges will apply.**


6. Check the cluster is up and running:

    ```
    # Execute the following to check the cluster's state
    $ kops validate cluster \
        --name <cluster-name> \
        --state s3://<bucket-name>

    # Execute the following to get some cluster's info (for example, the API's url).
    $ kubectl cluster-info
    ```

    **Note:** You can also access the bastion host through SSH.


7. [Optional] Install the [Kubernetes dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/).

    1. Edit the cluster (as you have done in step 4):

        ```
            $ kops edit cluster \
                --name <cluster-name> \
                --state s3://<bucket-name>
        ```    

    2. Add the following lines to the cluster manifest:

        ```yaml
        # ...
        spec:
          addons:
          - manifest: kubernetes-dashboard
        # ...
        ```

        **Note:** You can do this before building the cluster (as you did in step 5)


    3.  Rebuild the cluster (as you have done in step 5):

        ```
        $ kops edit cluster \
            --name <cluster-name> \
            --state s3://<bucket-name> \
            --yes
        ```

    4. You might have to perform a rolling update to see changes applied in the cluster:

         ```
        $ kops rolling-update cluster \
            --name <cluster-name> \
            --state s3://<bucket-name> \
            --yes
        ```

        This operation might take a while (several minutes)

    5. Once the rolling update has finished, validate the cluster:

        ```
        $ kops validate cluster \
            --name <cluster-name> \
            --state s3://<bucket-name>
        ```

    6. Access the Dashboard using your favorite browser. 

        The URL is:

        ```
        https://api.<cluster-name>/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
        ```

        You will be prompted for the username and password.

        - The username is `admin`

        - The password can be obtained using the following command:

            ```
            $ kubectl config view --minify
            ```

        Once you have logged in, you will be promted for kubeconfig file or a token. This is needed to grant permissions to the dashboard process. Execute the following commands to obtain the token:

        ```
        # First assing the cluster-admin role to the dashbaord service role
        $ kubectl create clusterrolebinding kubernetes-dashboard \
            -n kube-system \
            --clusterrole=cluster-admin \
            --serviceaccount=kube-system:kubernetes-dashboard 

        # Then get the token
        $ kubectl get secret \
            $(kubectl get serviceaccount kubernetes-dashboard \
                -o jsonpath="{.secrets[0].name}" \
                -n kube-system \
            ) \
           -o jsonpath="{.data.token}" \
               -n kube-system \
            | base64 --decode 
        ```

        Copy the output of the last command and enter it in the dashboard's UI.


8. [Optional] As having a provisioned cluster costs money, you might want to delete if you are just playing around. Execute the following command to terminate all AWS resources **created by KOPS**:

     ```
    $ kops delete cluster \
        --name <cluster-name> \
        --state s3://<bucket-name> \
        --yes
    ```
