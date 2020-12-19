# Configure Multi-tenancy in IBM Edge Application Manager

**Description**: This guide will walk through how to set up a new IEAM Organization.  Additional Organizations can be set up a multi-tenancy configuration. 

## Table of Contents
- [Pre-requisites](#pre-requisites)
- [Set up environment variables for hzn CLI](#set-up-environment-variables)
- [Configure a Hub Admin](#configure-a-hub-admin)
- [Create a new IEAM Organization](#create-a-new-ieam-organization)
- [Generate API Key to register Edge device](#generate-api-key-to-register-edge-device)


# Pre-requisites:
The instructions below assumes that the following conditions have been met.  
- An instance of OpenShift 4.5 has been provisioned.
- IBM Edge Application Manager Operator has been installed and an IEAM Hub has been created.
- IEAM Post Install Configuration tasks have been completed.  
- IEAM Hub has been configured to use OpenLDAP for authentication.


# Set up environment variables
It is required to be logged in to the OpenShift cluster where IEAM is installed.
```
export CLUSTER_URL=https://$(oc get cm management-ingress-ibmcloud-cluster-info -o jsonpath='{.data.cluster_ca_domain}')
export CLUSTER_USER=$(oc -n ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_username}' | base64 --decode)
export CLUSTER_PW=$(oc -n ibm-common-services get secret platform-auth-idp-credentials -o jsonpath='{.data.admin_password}' | base64 --decode)
oc --insecure-skip-tls-verify=true -n kube-public get secret ibmcloud-cluster-ca-cert -o jsonpath="{.data.ca\.crt}" | base64 --decode > ieam.crt
export HZN_MGMT_HUB_CERT_PATH="$PWD/ieam.crt"
export HZN_FSS_CSSURL=${CLUSTER_URL}/edge-css
```


# Configure a Hub Admin
## Login as cluster admin
```
# Retrieve IEAM Management Console:
echo https://$(oc get cm management-ingress-ibmcloud-cluster-info -o jsonpath='{.data.cluster_ca_domain}')/edge

# Retrieve "admin" password
oc get secrets -n ibm-common-services platform-auth-idp-credentials -o jsonpath='{.data.admin_password}' | base64 --decode && echo

cloudctl login -a https://$(oc get cm management-ingress-ibmcloud-cluster-info -o jsonpath='{.data.cluster_ca_domain}') -u admin -p <admin password> -n ibm-edge --skip-ssl-validation
```

## Create IAM account for Hub Admin
```
IAM_ACCOUNT_NAME='hub admin account'   
cloudctl iam account-create "$IAM_ACCOUNT_NAME" -d 'account for the hub admin users'

# Sample Output:
Name          hub admin account
Description   account for the hub admin users
ID            xxxx-xxxx

IAM_ACCOUNT_ID=$(cloudctl iam account "$IAM_ACCOUNT_NAME" | grep -E '^ID' | awk '{print $2}')
IAM_TEAM_ID=$(cloudctl iam teams -s | grep -m 1 "$IAM_ACCOUNT_NAME" | awk '{print $1}')
```

## Add "hubadmin1" OpenLDAP user into Hub Admin IAM account
```
HUB_ADMIN_USER=hubadmin1
cloudctl iam user-import -u $HUB_ADMIN_USER
cloudctl iam user-onboard $IAM_ACCOUNT_ID -r PRIMARY_OWNER -u $HUB_ADMIN_USER
IAM_RESOURCE_ID=$(cloudctl iam resources | grep ':n/ibm-edge:')
cloudctl iam resource-add $IAM_TEAM_ID -r $IAM_RESOURCE_ID

# Sample Output:
Resource crn:v1:icp:private:k8:mycluster:n/ibm-edge::: added
OK
```

## Set "hubadmin1" as Hub Admin
```
# Retrieve exchange root password
oc get secret ibm-edge-auth -o jsonpath='{.data.exchange-root-pass}' | base64 --decode && echo

EXCHANGE_ROOT_PW=<Exchange Root password>
export HZN_ORG_ID=root
export HZN_EXCHANGE_USER_AUTH=root/root:$EXCHANGE_ROOT_PW
export HZN_EXCHANGE_URL=https://$(oc get cm management-ingress-ibmcloud-cluster-info -o jsonpath='{.data.cluster_ca_domain}')/edge-exchange/v1
HUB_ADMIN_USER=hubadmin1

curl -sS --insecure -w %{http_code} -u "$HZN_EXCHANGE_USER_AUTH" -X POST -H Content-Type:application/json -d '{"hubAdmin":true,"password":"","email":""}' $HZN_EXCHANGE_URL/orgs/root/users/$HUB_ADMIN_USER | jq

# Sample Output:
{
  "code": "ok",
  "msg": "1 user added successfully"
}
201

# If user exists already, use "-X PUT" instead in the curl request
curl -sS --insecure -w %{http_code} -u "$HZN_EXCHANGE_USER_AUTH" -X PUT -H Content-Type:application/json -d '{"hubAdmin":true,"password":"","email":""}' $HZN_EXCHANGE_URL/orgs/root/users/$HUB_ADMIN_USER | jq

# Sample Output:
{
  "code": "ok",
  "msg": "user updated successfully"
}
201
```

## Verify hubadmin1 user is a "hubAdmin"
```
hzn exchange user list $HUB_ADMIN_USER

# Sample Output:
{
  "root/hubadmin1": {
    "admin": false,
    "email": "",
    "hubAdmin": true,
    "lastUpdated": "2020-12-19T16:36:55.524Z[UTC]",
    "password": "<password>",
    "updatedBy": "root/root"
  }
}
```


# Create a new IEAM Organization
##Login as hub admin user (hubadmin1)
```
cloudctl login -a https://$(oc get cm management-ingress-ibmcloud-cluster-info -o jsonpath='{.data.cluster_ca_domain}') -u hubadmin1 -p <hub admin password> -n ibm-edge --skip-ssl-validation
```

## Generate an API key for Hub Admin User
```
cloudctl iam api-key-create "${HUB_ADMIN_USER}-api-key" -d "API key for $HUB_ADMIN_USER"

# Sample Oputput
Creating API key hubadmin1-api-key as hubadmin1...
OK
API key hubadmin1-api-key created

Please preserve the API key! It cannot be retrieved after it's created.

Name          hubadmin1-api-key
Description   API key for hubadmin1
Bound To      crn:v1:icp:private:iam-identity:::IBMid:user:hubadmin1
Created At    2020-12-19T16:43+0000
API Key       <API Key>

# Set the following environment variables and verify the API key is functional
HUB_ADMIN_API_KEY=<API Key>
export HZN_ORG_ID=root
export HZN_EXCHANGE_USER_AUTH=root/iamapikey:$HUB_ADMIN_API_KEY
hzn exchange org list -o root

# Sample Output:
[
  "root",
  "IBM",
  "edge-sample-org"
]
```

## Create an organization
```
HUB_ADMIN_API_KEY=<API Key>
# Log in as cluster administrator to create an IAM account for the new IEAM Org
cloudctl login -a https://$(oc get cm management-ingress-ibmcloud-cluster-info -o jsonpath='{.data.cluster_ca_domain}') -u admin -p <admin password> -n ibm-edge --skip-ssl-validation


NEW_ORG_ID=edge-org-a
IAM_ACCOUNT_NAME="$NEW_ORG_ID"
cloudctl iam account-create "$IAM_ACCOUNT_NAME" -d "$IAM_ACCOUNT_NAME account"

# Sample Output:
Name          edge-org-a
Description   edge-org-a account
ID            xxxx-xxxx


IAM_ACCOUNT_ID=$(cloudctl iam account "$IAM_ACCOUNT_NAME" | grep -E '^ID' | awk '{print $2}')
IAM_TEAM_ID=$(cloudctl iam teams -s | grep -m 1 "$IAM_ACCOUNT_NAME" | awk '{print $1}')

# Set the user to be assigned as Organization Administrator
ORG_ADMIN_USER=admin1
cloudctl iam user-onboard $IAM_ACCOUNT_ID -r PRIMARY_OWNER -u $ORG_ADMIN_USER
IAM_RESOURCE_ID=$(cloudctl iam resources | grep ':n/ibm-edge:')
cloudctl iam resource-add $IAM_TEAM_ID -r $IAM_RESOURCE_ID

# Sample Output:
Resource crn:v1:icp:private:k8:mycluster:n/ibm-edge::: added
OK

# Create IEAM Org
export HZN_ORG_ID=root
export HZN_EXCHANGE_USER_AUTH=root/iamapikey:$HUB_ADMIN_API_KEY
export HZN_EXCHANGE_URL=https://$(oc get cm management-ingress-ibmcloud-cluster-info -o jsonpath='{.data.cluster_ca_domain}')/edge-exchange/v1
hzn exchange org create -a IBM/agbot -t "cloud_id=$IAM_ACCOUNT_ID" --description "$NEW_ORG_ID organization" $NEW_ORG_ID

# Sample Output:
Organization edge-org-a is successfully added to the Exchange.
Agbot IBM/agbot is responsible for deploying services in org edge-org-a

hzn exchange agbot addpattern IBM/agbot IBM '*' $NEW_ORG_ID      
hzn exchange org list $NEW_ORG_ID

# Sample Output:
{
  "edge-org-a": {
    "description": "edge-org-a organization",
    "heartbeatIntervals": {
      "intervalAdjustment": 0,
      "maxInterval": 0,
      "minInterval": 0
    },
    "label": "edge-org-a",
    "lastUpdated": "2020-12-19T17:01:03.537Z[UTC]",
    "limits": {
      "maxNodes": 0
    },
    "orgType": "",
    "tags": {
      "cloud_id": "e0e8-1c03"
    }
  }
}

# Add IEAM Organization Administrator to Horizon Exchange
hzn exchange user create -o $NEW_ORG_ID -A $ORG_ADMIN_USER "" "admin1@email.com"
hzn exchange user list -o $NEW_ORG_ID $ORG_ADMIN_USER

# Sample Output:
{
  "edge-org-a/admin1": {
    "admin": true,
    "email": "admin1@email.com",
    "hubAdmin": false,
    "lastUpdated": "2020-12-19T17:04:07.600Z[UTC]",
    "password": "<Password>",
    "updatedBy": "root/hubadmin1"
  }
}
```


# Generate API Key to register Edge device
## Generate API Key as Org Admin
```
# Log in as Organization Administrator
cloudctl login -a https://$(oc get cm management-ingress-ibmcloud-cluster-info -o jsonpath='{.data.cluster_ca_domain}') -u admin1 -p <admin1 password> -c $IAM_ACCOUNT_ID --skip-ssl-validation

# Create API Key
cloudctl iam api-key-create "${ORG_ADMIN_USER}-api-key" -d "API key for $ORG_ADMIN_USER"

# Sample Ouptut:
Creating API key admin1-api-key as admin1...
OK
API key admin1-api-key created

Please preserve the API key! It cannot be retrieved after it's created.

Name          admin1-api-key
Description   API key for admin1
Bound To      crn:v1:icp:private:iam-identity:::IBMid:user:admin1
Created At    2020-12-19T17:08+0000
API Key       <API Key>

# Set API Key for Organization Administrator
# Tip: If you add this user to additional IAM accounts in the future, you do not need to create an API key for each account. The same API key will work in all IAM accounts this user is a member of, and therefore in all IEAM organizations this user is a member of.
ORG_ADMIN_API_KEY=<API Key>

export HZN_ORG_ID=edge-org-a
export HZN_EXCHANGE_USER_AUTH=$HZN_ORG_ID/iamapikey:$ORG_ADMIN_API_KEY
# Verify API key is functional
hzn exchange user list

# Sample Output:
{
  "edge-org-a/admin1": {
    "admin": true,
    "email": "admin1@email.com",
    "hubAdmin": false,
    "lastUpdated": "2020-12-19T17:04:07.600Z[UTC]",
    "password": "********",
    "updatedBy": "root/hubadmin1"
  }
}
```
