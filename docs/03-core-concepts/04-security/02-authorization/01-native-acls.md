---
sidebar_label: Native ACLs
---


# Native ACLs

Ozone provides a native access control list (ACL) support which can be used without the need to introduce a separate IAM infrastructure (e.g. Apache Ranger). The Ozone native ACL tries to cater for both POSIX ACL semantics and S3 ACL. This documentation describes the Ozone Native ACL concepts and its usage.

:::info
For Ranger ACL documentation, please refer to this [page](02-ranger-acls.md).
:::

## Pre-Requisites

A basic understanding of Ozone concepts such as Ozone namespace objects (e.g. volume, bucket, and key) is required. It is also recommended to have some knowledge in POSIX and S3 permission model. Please refer to the relevant documentations to get familiar with these concepts.

## Overview

To give a brief overview, whenever an Ozone client makes a request to the Ozone cluster with ACL enabled, the client first needs to authenticate the request to get the client's credentials (username and groups). These information are included in the request's context and is used by the Ozone cluster to authorize the request.

An *object* can be attached by one or more Ozone ACLs which contains information about *who* is authorized to operate on the *object* using which *right*

To give a brief overview, an Ozone ACL is attached to an *Object* which species which *who* is authorized to operate on the *Object* using which *right*. All client request contains a `UserGroupInformation` (UGI) used to decide *who* is accessing the objects. The UGI is checked against the object's ACLs (and parent ACLs if necessary) based on the User's / Group's name for certain request-specific *rights*. For instance, to authorize a *Create Key* request, the parent bucket's ACL has to contain an ACL entry that contains the user's name / group's name and has the WRITE right.

## Enabling Ozone Native ACLs

To enable the Ozone Native ACLs support, add the following properties to `ozone-site.xml`.

```xml title="ozone-site.xml"
<configuration>
  <property>
    <name>ozone.acl.enabled	</name>
    <value>true</value>
  </property>
  <property>
    <name>org.apache.hadoop.ozone.security.acl.OzoneNativeAuthorizer</name>
    <value></value>
  </property>
</configuration>
```

## Ozone ACL

### Basic Ozone ACL concepts

The basic Ozone ACL concepts are:

1. **Object**
2. **Scope**
3. **Identity Type**
4. **Right**

The following sections cover these concepts.

#### Object

The object in the context of Native ACL refers to any object that is recognized by the Native ACL permission model. The Native ACL authorizer uses the object to see which resource the request is trying to access. These objects are the only recognized objects which can be attached by an ACL.

1. **Volume**: An Ozone volume (e.g. `/volume`)
2. **Bucket**: An Ozone bucket (e.g. `/volume/bucket`)
3. **Key**: An Ozone key (e.g. `/volume/bucket/key`)
4. **Prefix**: A prefix under a bucket (e.g. `/volume/bucket/prefix/`)

Volume, bucket, and key should be familiar as it maps directly to the Ozone namespace objects. Prefix is an object specific to Ozone ACL Model and should not to be confused by directory. Prefix ACL is covered in a separate [section](#prefix-acl) since it is a new concept and it is not a part of the standard POSIX / S3 ACL models.

Each object can be attached with one or more unique Ozone ACLs which are retrieved (along with the object itself) when a request is trying to access the object.

#### Scope

Similar to POSIX ACL, each Ozone ACL can be categorized to one of two scopes:

1. `ACCESS`: Applied only to the specific object and not inheritable.
2. `DEFAULT`: Applied to the specific object and will be inherited by object's descendants.

ACL scope can be thought of as the "type" of ACL. From the descriptions, the difference between `ACCESS` and `DEFAULT` scope is that `DEFAULT` ACLs are inherited whereas `ACCESS` ACLs are not.

The default inheritance logic of `DEFAULT` ACLs is [following section](#acl-inheritance)

#### Identity Type

This is the *who* of the permission model. These are the current recognized identity types:

1. `USER`
2. `GROUP`
3. `WORLD`
4. `ANONYMOUS`
5. `CLIENT_IP`

#### Right

Each Ozone ACL is attached with one or more ACL rights which explains the types of operation a user can perform for the particular object.

These are the ACL rights in Ozone:

1. `READ`
2. `WRITE`
3. `CREATE`
4. `LIST`
5. `DELETE`
6. `READ_ACL`
7. `WRITE_ACL`
8. `ALL`
9. `NONE`

Each type client request (e.g. create key, list buckets, etc) requires a different set of rights for different objects. For example, creating a key requires the user to have a `WRITE` right both to the parent bucket and the volume.

Please refer to the access matrix for all the required ACLs for different requests.

## Ozone Native ACL Permission Model

Ozone ACL are one of the resource-based authorization models which can be used to manage access to objects in the Ozone cluster.

### Authenticating Request

Before the authorization process of Ozone client request, the Ozone cluster first need to authenticate the identity of the user making the request. Ozone adopts Hadoop's extrinsic approach to retrieve user identity (e.g. Simple, Kerberos, LDAP, etc). After the user's username and group information are retrieved, they are encapsulated in the `UserGroupInformation`. The username and the group information are then included in the request's context and used by the Ozone cluster to authorize the request.

:::info
A S3 user accessing Ozone via AWS v3 signature protocol will be translated to the appropriate Kerberos user by Ozone Manager.
:::




### Prefix ACL

### ACL Inheritance

### ACL initial ACLs

### ACL Permission Flowchart

## Usage

To enable Ozone Native ACL, add the following properties to the `ozone-site.xml`.

| Property                   | Value                                                      |
| -------------------------- | ---------------------------------------------------------- |
| ozone.acl.enabled	         | true                                                       |
| ozone.acl.authorizer.class | org.apache.hadoop.ozone.security.acl.OzoneNativeAuthorizer |

### CLI

### Java API

## Limitations

## Future Works