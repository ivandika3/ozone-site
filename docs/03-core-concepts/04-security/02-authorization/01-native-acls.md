---
sidebar_label: Native ACLs
---


# Native ACLs

Ozone provides a native access control list (ACL) support which can be used without the need to introduce a separate IAM infrastructure (e.g. Apache Ranger). The Ozone native ACL tries to cater for both POSIX ACL semantics and S3 ACL. This documentation describes the Ozone Native ACL concepts and its usage.

Ozone Native ACL is a resource-based permission model, 

:::info
For Ranger ACL documentation, please refer to this [page](./02-ranger-acls.md).
:::

## Pre-Requisites

A basic understanding of Ozone concepts such as Ozone namespace objects (e.g. volume, bucket, and key) is required. It is also recommended (but not necessary) to have some knowledge in POSIX and S3 permission model. Please refer to the relevant documentations to get familiar with these concepts.

## Overview

Whenever an Ozone client makes a request to the Ozone cluster with ACL enabled, the client first needs to authenticate the request to get the client's credentials (username and groups). These information are included in the request's context and is used by the Ozone cluster to authorize the request.

An *object* can be attached by one or more Ozone ACLs which contains information about *who* is authorized to operate on the *object* using which *right*

To give a brief overview, an Ozone ACL is attached to an *Object* which species which *who* is authorized to operate on the *Object* using which *right*. All client request contains a `UserGroupInformation` (UGI) used to decide *who* is accessing the objects. The UGI is checked against the object's ACLs (and parent ACLs if necessary) based on the User's / Group's name for certain request-specific *rights*. For instance, to authorize a *Create Key* request, the parent bucket's ACL has to contain an ACL entry that contains the user's name / group's name and has the WRITE right.

## Enabling Ozone Native ACLs

To enable the Ozone Native ACLs support, add the following properties to `ozone-site.xml`.

```xml title="ozone-site.xml"
<configuration>
  <property>
    <name>ozone.acl.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>ozone.acl.authorizer.class</name>
    <value>org.apache.hadoop.ozone.security.acl.OzoneNativeAuthorizer</value>
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

The default inheritance logic of `DEFAULT` ACLs is [following section](#default-acl-inheritance)

#### Identity Type

This is the *who* of the permission model.

These are the current recognized identity types:

1. `USER`
2. `GROUP`
3. `WORLD`
4. `ANONYMOUS`
5. `CLIENT_IP`

An Ozone object can contain multiple ACLs with different identity type. When authenticating user's request against the the object's ACLs, the authorizer checks ACL's identity type and based on that it will use different part of the user's UGI to authenticate the request. For example, if the ACL has identity type `USER`, the authorizer retrieves the user's username (i.e. `UserGroupInformation#getShortUserName`) and if the ACL has identity type `GROUP`, the authorizer retrieves the user's group(s) (i.e. `UserGroupInformation#getGroupNames`). An ACL with `WORLD` identity type applies to all users so should be used with caution.

:::info
Currently `USER`, `GROUP`, and `WORLD` identity types are mainly used. `ANONYMOUS` and `CLIENT_IP` have not been mapped and currently has the behavior of `WORLD`.
:::

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

### Authenticating Request

Before the authorization process of Ozone client request, the Ozone cluster first need to authenticate the identity of the user making the request. Ozone adopts Hadoop's extrinsic approach to retrieve user identity (e.g. Simple, Kerberos, LDAP, etc). After the user's username and group information are retrieved, they are encapsulated in the `UserGroupInformation`. The username and the group information are then included in the request's context and used by the Ozone cluster to authorize the request.

A `RequestContext` is a context object that is created during each OM request. It encapsulates ACL-related information, such as the `UserGroupInformation` of the user and ACL rights (e.g. `READ`, `WRITE`) required to access the object for this request. An `OzoneObject` is also created to represent what Ozone namespace object the request is trying to access. Both of these objects are passed to the `checkAccess` method of `IAccessAuthorizer` (e.g. `OzoneNativeAuthorizer`) instance that uses it the ACL Model logic, such as [deriving the parent access right](#parent-child-relationship). The authorizer then delegates each namespace object's the `checkAccess` method of to their own managers (e.g. `VolumeManager`, `BucketManager`, `KeyManager`, `PrefixManager`) for object type-specific logic.

:::info
A S3 user accessing Ozone via AWS v3 signature protocol will be translated to the appropriate Kerberos user by Ozone Manager.
:::

## Ozone Native ACL Permission Model

The following sections highlight important concepts necessary for the Ozone Native ACL permission model.

### Owners

Currently Ozone Native ACL currently supports [volume owner](../../03-namespace/01-volumes/02-owners.md) and [bucket owner](../../03-namespace/02-buckets/02-owners.md).

### Prefix ACL

### Linked Bucket

### Ozone Administrators

Ozone provides a way to specify administrator and read-only administrator configuration for users and groups in the `ozone-site.xml` configuration.

* Ozone administrators bypass all the checks in the `OzoneNativeAuthorizer`
* Ozone readonly administrators bypass only read operations (i.e. `READ`, `READ_ACL`, and `LIST`). Write operations still need to be checked.

The following tables contains the Ozone admin related properties to be set in `ozone-site.xml`:

| Property                | Description                                              |
| ----------------------------- | -------------------------------------------------------- |
| `ozone.administrators`        | Ozone administrators. If not set, only the user who launches an Ozone service will be the admin user  |
| `ozone.administrators.groups` | Ozone administrator groups |
| `ozone.readonly.administrators` | Ozone read only admin users |
| `ozone.readonly.administrators.groups` | Ozone read only admin groups  |

Note that each configuration are comma-delimited.

:::warning
`ozone.administrators` or `ozone.administrators.groups` properties must be set if Ozone services (e.g. OM, SCM, and Datanode) are start by different users. Otherwise, the RPC layer will reject calls from other servers which are started by users not in the list.
:::

### User Default ACL Rights

On top of the inherited ACLs (if any), when a user creates a new volume / bucket / keys, default USER and GROUP ACLs of the user are automatically created based on the user's `UserGroupInformation`. The ACL rights of these default ACLs are governed by these configurations:

| Property                | Default Value   | Description                                              |
| ----------------------- | --------------- | -------------------------------------------------------- |
| `ozone.om.user.rights`  | `ALL`           | Default user permissions set for a newly created object  |
| `ozone.om.group.rights` | `ALL`           | Default group permissions set for a newly created object |

This means that user that created the object and other users in the same group as the user created are able to have the specified rights for the created object.

There are some special cases:

* For a linked bucket, a `world::rw` permission is created to allow all request to resolve the bucket link to pass.
  * This is similar to the POSIX Symbolic link behavior.  
* If a bucket or key is created through S3 API (`create-bucket` or `put-object`), the group ACL rights will be overridden to `NONE`, regardless of the `ozone.om.group.rights` configuration.
  * Unlike POSIX permission model, it is not intuitive to grant users in the group rights to a newly created object

### Permission Flowchart

### Parent Child Relationship

In Ozone Native ACL Model, Ozone namespace objects are a part of object hierarchy. Currently Ozone Native ACL model object hierarchy is:

* Volume is a direct parent of Bucket
* Bucket is a direct parent of either Prefix or Key object
* Prefix is a direct parent of Key object

This "direct parent" relationship is transitive. Meaning that although Volume is a direct parent of Bucket, it can also be parent of Prefix and Key object. Whether an Ozone object is a parent of another Ozone object can be attributed whether the child Ozone object path components are prefixed by the parent Ozone object's.

The object hierarchy is important because Ozone Native ACL requires user's operation on an Ozone object to check whether the user has permission to access the parent objects. The parent check goes up all the way to the root parent (i.e. Volume). For example, accessing a key requires to check whether a user has an access to parent prefix (if any), parent bucket, and parent volume. However, access right required on the parent, compared to the child object is different. For example, a delete operation on a child object `DELETE` right on the child object but only `READ` right on the parent bucket and parent volume.

The following tables shows how these "derived" rights are calculated for bucket, keys, and prefix objects. Note that the check is recursive until there is no more parent object.

#### Write-related operations

| Operation   | Child       | Parent                                                                     |
| ----------  | ----------- | -------------------------------------------------------------------------- |
| `CREATE`    | None        | `WRITE`                                                                    |
| `DELETE`    | `DELETE`    | `READ`                                                                     |
| `WRITE`     | `WRITE`     | `WRITE` (for key or prefix, only requires `READ` access on parent volume)  |
| `WRITE_ACL` | `WRITE_ACL` | `READ`                                                                     |

Note that a special case on derived parent volume right for `WRITE` operation on prefix and key is `READ` instead of `WRITE`.

#### Read-related operations

| Operation | Child     | Parent |
| --------- | --------- | ------ |
| `READ`    | `READ`    | `READ` |
| `LIST`    | `LIST`    | `READ` |
| `READ_ACL`| `READ_ACL`| `READ` |

### DEFAULT ACL Inheritance



### ACL Access Tables

This section provides a table for the ACL rights required for namespace object hierarchies for different Ozone operations. For example, in order to lookup a key, a user is required to have a `READ` access to a key, `READ` access to the parent prefix (if any), `READ` access to parent bucket, and `READ` access to parent bucket.

The following access tables assume the the user trying to access is not an Ozone administrator / read-only administrator and it is not an owner of any of the namespace object (for example, volume and bucket)

* For Ozone administrator concept, please take a look at the [administrator section](#ozone-administrators)
* For owner concept, please take a look at the [owner section](#owners)

For example, the "Create volume" operation indicates that there is no applicable volume right for a user to create a volume. This implies that only an admin can create a volume (since an admin is able to do virtually any operation). Similarly, if a user is accessing a key under a bucket owns by the user, the access is granted immediately.

#### Volume Level Operations

| Operation               | Volume Right                            |
| ----------------------- | --------------------------------------- |
| Create volume           | -                                       |
| List volumes            |  `ozone.om.volume.listall.allowed=true` |
| Get volume info         | `READ`                                  |
| Delete volume           | `DELETE`                                |
| Set quota               | `WRITE`                                 |
| Read volume ACL         | `READ_ACL`                              |
| Write volume ACL        | `WRITE_ACL`                             |
| Set volume owner        | `WRITE_ACL`                             |
| Tenant creation         | `WRITE_ACL`                             |
| Tenant deletion         | `WRITE_ACL`                             |

#### Bucket Level Operations

| Operation                                                                                 | Volume Right                            | Bucket Right                            |
| ----------------------------------------------------------------------------------------- | --------------------------------------- | --------------------------------------- |
| Create bucket                                                                             | `WRITE`                                 | None                                    |
| List buckets                                                                              | `LIST`                                  | None                                    |
| Get bucket info                                                                           | `READ`                                  | `READ`                                  |
| Resolve bucket link                                                                       | `READ`                                  | `READ`                                  |
| Delete bucket                                                                             | `READ`                                  | `DELETE`                                |
| List snapshot                                                                             | `READ`                                  | `LIST`                                  |
| List trash                                                                                | `READ`                                  | `LIST`                                  |
| Set bucket property (quota, storage type, versioning enabled, replication config)         | `READ`                                  | -                                       |
| Recover trash                                                                             | `READ`                                  | `WRITE`                                 |
| Read bucket ACL                                                                           | `READ`                                  | `READ_ACL`                              |
| Write bucket ACL                                                                          | `READ`                                  | `WRITE_ACL`                             |
| Set bucket owner                                                                          | `READ`                                  | `WRITE_ACL`                             |
| List keys                                                                                 | `READ`                                  | `LIST`                                  |
| List status (on bucket)                                                                   | `READ`                                  | `READ`                                  |

Note:

* Resolve bucket link operation access check is done for every operation on linked bucket or prefix / key under the linked bucket

### Prefix Level Operations

| Operation                  | Volume Right                            | Bucket Right                            | Prefix Right                          |
| ---------------------------| --------------------------------------- | --------------------------------------- | ------------------------------------- |
| Read prefix ACL            | `LIST`                                  |                                         |                                       |
| Write prefix ACL           | `READ`                                  |                                         |                                       |

#### Key / File / Directory Level Operations

| Operation                             | Volume Right                            | Bucket Right                            | Prefix Right                          | Open Key Right (if any)     | Key Right                |
| ------------------------------------- | --------------------------------------- | --------------------------------------- | ------------------------------------- | --------------------------- | ------------------------ |
| Create key / directory / file                             |  `READ`                                  | `WRITE`                                          |                                       |                             |                          |
| Allocate block                             |  `READ`                                  | `WRITE`                                          |                                       |                             |                          |
| Commit key                             |  `READ`                                  | `WRITE`                                          |                                       |                             |                          |
| Rename key                             |  `READ`                                  | `WRITE`                                          |                                       |                             |                          |
| Copy object                             |  `READ`                                  | `WRITE`                                          |                                       |                             |                          |
| Delete key                            | `LIST`                                  |                                         |                                       |                             |                          |
| Lookup / Read key                              | `READ`                                  | `READ`                                  |                                       |                             |                          |
| List status (on directory / key)      | `READ`                                  | `READ`                                  |                                       |                             |                          |
| Write key ACL              | `READ`                                  | `DELETE`                                |                                       |                             |                          |
| Read key ACL              | `READ`                                  | `LIST`                                  |                                       |                             |                          |
| Write key ACL                 | `READ`                                  | `LIST`                                  |                                       |                             |                          |
| Set bucket property        | `READ`                                  | -                                       |                                       |                             |                          |
| Recover trash              | `READ`                                  | `WRITE`                                 |                                       |                             |                          |
| Read bucket ACL            | `READ`                                  | `READ_ACL`                              |                                       |                             |                          |
| Write bucket ACL           | `READ`                                  | `WRITE_ACL`                             |                                       |                             |                          |
| Set bucket owner           | `READ`                                  | `WRITE_ACL`                             |                                       |                             |                          |
| List keys                  | `READ`                                  | `LIST`                                  |                                       |                             |                          |
| List status (on bucket)    | `READ`                                  | `READ`                                  |                                       |                             |                          |

`OMFileRequest#hasChildren`

## S3 ACL Support


## Usage

### CLI

### Java API

## Limitations

## Future Ideas

### Directory ACL as part of Ozone Native ACL

### Recursive Prefix ACL Check

### Deprecating Key ACL

### Support IP and Anonymous ACL right 