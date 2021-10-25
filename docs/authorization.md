# Authorization

## Policies

Mainflux uses policies to control permissions on entities: **users**, **things**, and **groups**. Under the hood, Mainflux uses [ORY Keto](https://www.ory.sh/keto/) that is an open-source implementation of ["Zanzibar: Google's Consistent, Global Authorization System"](https://www.usenix.org/conference/atc19/presentation/pang)

Policies define permissions for the entities. For example, *which user* has *access* to *a specific thing*. Such policies have three main components: **subject**, **object**, and **relation**.

To put it briefly: 

**Subject**: As the name suggests, it is the subject that will have the policy. Mainflux uses entity UUID on behalf of the real entities.

**Object**: Objects are Mainflux entities (e.g. *thing* or *group*) represented by their UUID.

**Relation**: This is the action that the subject wants to do on the object.

> For more conceptual details, you can refer [official ORY Keto documentation](https://www.ory.sh/keto/docs)

All three components create a single policy.

For example, let's assume we have a following policy: `"user_id_123" has "read" relation on "thing_id_123"`. This policy means that subject (a user with ID: `user_id_123`) has a relation (`read`) on the object (a thing with ID: `thing_id_123`). Based upon this example, If the user wants to view a `Thing`, Mainflux first identifies the user with Authentication Keys and checks the policy as: 
```
User with ID: `user_id_123` has `read` relation on the thing with ID: `thing_id_123`. 
```
If the user has no such policy, the operation will be denied; otherwise, the operation will be allowed. In this case, since the user `user_id_123` has the policy, the `read` operation on the thing `thing_id_123` will be allowed for the user with ID `user_id_123`. On the other hand, requests coming from other users (who have a different ID than `user_id_123`) will be denied.

In order to check whether a user has the policy or not, Mainflux makes a gRPC call to Keto API, then Keto handles the checking existence of the policy. 

All policies are stored in the Keto Database. The database responsible for storing all policies is deployed along with the Mainflux, as a standalone PostgreSQL database container.

## Predefined Policies

Mainflux comes with predefined policies.

### Users service related policies

- By default, Mainflux allows anybody to create a user. If you disable this default policy, only *admin* is able to create a user.
This default policy can be disabled through an environment variable called `MF_ONLY_ADMIN_CREATES_USER` in deployment time. 
Mainflux creates a special policy to enable this feature as follows: `user#create@*`. This policy dictates that subject `*` has `create` relation on the object `users`. Here, Mainflux uses a special `*` subject to represent all users. If this policy is defined, everybody can create new users.
- All users are a `member of the users`. To be more precise, once the new user is created, the policy service creates the following policy: 
`users#member@<user_id>` indicating that the subject `<user_id`> has `member` relation on the object `users`.
- The admin has a special policy indicating that the user is admin. This policy is the following: 
`<admin_id>` has `member` relation on the object `authorities`. 

### Things service related policies

- There are 3 policies regarding `Things`: `read`, `write` and `delete`.
- When a user creates a thing, the user will have `read`, `write` and `delete` policies on the `Thing`.
- In order to view a thing, you need `read` policy on that thing.
- In order to update and share the thing, you need a `write` policy on that thing.
- In order to remove a thing, you need a `delete` policy on that thing.

### Group entity related policies

- Once the user creates a new group, the user will have a `member` policy on the group.
- If you assign a new User member to your Users Group, the new user will have a `member` policy on this particular Users Group.
- If you assign a new Thing member to your Things Group, whatever has `access` policy on the Things Group will have `read`, `write` and `delete` policy on the Things defined in the Thing Group.
- Mainflux allows users to assign access rights of the Things group with the Users group. Thus, each member of the User group can access Things defined in the Thing group. In order to do so, the Policy service adds an `access` policy of the Thing Group for the users who have a `member` policy on the Users group. Therefore, the Users group members have `read`, `write` and `delete` policy on the Things defined in the Thing Group.

### Summary of the Defined Policies
- **`member`**: Identifies registered user's role such as `admin`. Also, it indicates memberships on the Group entity.
- **`read`, `write` and `delete`**: Controls access control for the Things.
- **`access`**: Controls whether you have an `access` to other group or not.
- **`create`**: Mainflux uses special `create` policy to allow everybody to create new users. If you want to enable this feature through the HTTP, you need to make following request:
```bash
curl -isSX POST http://localhost/policy -d '{"subjects":["*"],"policies": ["create"], "object": "user"}' -H "Authorization: <admin_auth_token>" -H 'Content-Type: application/json'
```

### Adding custom policies

You can add custom policies as well through an HTTP endpoint. *Only* admin can use this endpoint. Therefore, you need an authentication token for the admin.

**Caveat:** Only policies defined under [Summary of the Defined Policies](#summary-of-the-defined-policies) are allowed. Other policies are not allowed. For example, you can add `member` policy but not `custom-member` policy because `custom-member` policy is not defined on the system.

```bash
curl -isSX POST http://localhost/policy -d '{"subjects":["<user_id>"],"policies": ["member"], "object": "users"}' -H "Authorization: <admin_auth_token>" -H 'Content-Type: application/json'
```

## Example usage of adding a policy

Suppose we are using the Mainflux version that doesn't have a policies feature yet. Once you migrate a new version of the Mainflux including the Policy feature, your users will face a lack of authorization. For example, there is a user created before the Policy feature. This user is authenticated by `<user_auth_token`>. Although the following operation is valid, the user will have an authorization error.
```bash
mainflux-cli things create '{"name":"user-thing"}' <user_auth_token>

error: failed to create entity: 403 Forbidden
```
The reason is that the user has not enough policy to create a new Thing after migration. In order to create a new thing, the user has to have a `member` relation on the `users` key. So that, Mainflux understands that the requester user is authorized to create new Things.

The easiest solution for this problem is adding policies for the users through the HTTP endpoint. As described above, the user needs a `member` relation on the `users`. 
```bash
curl -isSX POST http://localhost/policy -d '{"subjects":["<user_id>"],"policies": ["member"], "object": "users"}' -H "Authorization: <user_auth_token> " -H 'Content-Type: application/json' 
```

So what this request does is add new policies for the subject defined in the `subjects` field of the request body. Henceforth, the subject (here `<user_id>`) will have a `member` relation on the object `users`. This policy allows the user to create new Things.

Please, keep in mind that this endpoint requires you to use `<user_auth_token>`, not any token. 

## Example usage of sharing a Thing

Let's assume, we have two users (called `user1` and `user2`) registered on the system who have `user_id_1` and `user_id_2` as their ID respectively.
Let's create a thing with the following command:

```bash
mainflux-cli things create '{"name":"user1-thing"}' <user1_auth_token>           

created: a1109d52-6281-410e-93ae-38ba7daa9381
```

This command creates a thing called `"user1-thing"` with ID = `a1109d52-6281-410e-93ae-38ba7daa9381`. Mainflux identifies the `user1` by using the `<user1_auth_token>`. After identifying the requester as `user1`, the Policy service adds `read`, `write` and `delete` policies to `user1` on `"user1-thing"`.


If `user2` wants to view the `"user1-thing"`, the request will be denied.

```bash
mainflux-cli things get a1109d52-6281-410e-93ae-38ba7daa9381 <user2_auth_token>

error: failed to fetch entity : 403 Forbidden
```

After identifying the requester as `user2`, the Policy service checks that `Is user2 allowed to view the "user1-thing"?` Since `user2` has no such policy (`read` policy on `"user1-thing"`), the Policy service denies this request.

Now, `user1` wants to share the `"user1-thing"` with `user2`. `user1` can achieve this via HTTP endpoint for sharing things as follows:

```bash
curl -isSX POST http://localhost/things/a1109d52-6281-410e-93ae-38ba7daa9381/share -d '{"user_ids":["<user2_id>]", "policies": ["read", "delete"]}' -H "Authorization: <user1_auth_token>" -H 'Content-Type: application/json'

HTTP/1.1 200 OK
Server: nginx/1.20.0
Date: Thu, 09 Sep 2021 11:36:10 GMT
Content-Type: application/json
Content-Length: 3
Connection: keep-alive
Access-Control-Expose-Headers: Location

{}
```

> Note: Since sharing a thing requires a `write` policy on the thing, `user2` cannot assign a new policy for `"user1-thing"` by itself.

Now, `user2` has `read` and `delete` policies on `"user1-thing"` which allows `user2` to view and delete `"user1-thing"`. However, `user2` cannot update the `"user1-thing"` because `user2` has no `write` policy on `"user1-thing"`.

Let's try again viewing the `"user1-thing"` as `user2`:
```bash
mainflux-cli things get a1109d52-6281-410e-93ae-38ba7daa9381 <user2_auth_token>

{
  "id": "a1109d52-6281-410e-93ae-38ba7daa9381",
  "key": "6c9c2146-de49-460d-8f0d-adce4ad37500",
  "name": "user1-thing"
}
```

As we expected, the operation is successfully done. The policy server checked that `Is user2 allowed to view "user1-thing"?` Since `user2` has a `read` policy on `"user1-thing"`, the Policy server allows this request. 

## Example usage of sharing entities via Group

Mainflux allows you to group entities (e.g., `Users` and `Things`) through `Group` object in `auth` service. You can find more details about usage of the `Group` at [Groups documentation.](groups.md)

In this example, we will demonstrate how you can share access of the Users group to the Things group. So, each member of the Users group will have access to each Thing assigned to the Things group. We are going to start from a clean Mainflux setup and follow these steps:

1. Create a new user and multiple Things,
2. Create a Thing and User group, and assign members to groups,
3. Share access of the groups

First of all, obtain a token for the default admin. You can use any user but for the simplicity of the document, the default admin will be used. 

> By default, Mainflux uses credentials described in [.env](https://github.com/mainflux/mainflux/blob/master/docker/.env#L46) for the default admin.


```bash
$ curl -s -S -i -X POST -H "Content-Type: application/json" http://localhost/tokens -d '{"email":"admin@example.com",
"password":"12345678"}'
HTTP/1.1 201 Created
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 08:26:45 GMT
Content-Type: application/json
Content-Length: 285
Connection: keep-alive
Strict-Transport-Security: max-age=63072000; includeSubdomains
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: *
Access-Control-Allow-Headers: *

{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MzQxNDk2MDUsImlhdCI6MTYzNDExMzYwNSwiaXNzIjoibWFpbmZsdXguYXV0aCIsInN1YiI6ImFkbWluQGV4YW1wbGUuY29tIiwiaXNzdWVyX2lkIjoiOTA3MjkzMDMtZDMwZC00YmQ5LTkwMTYtNDljMThjZmY4YjUxIiwidHlwZSI6MH0.G1kjXiGX76BqpytmLdXtjLF9s9K5CVm4ScNMIaKlkwE"}
```

You can store the generated token because we will need it in further steps.

```bash
$ export token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MzQxNDk2MDUsImlhdCI6MTYzNDExMzYwNSwiaXNzIjoibWFpbmZsdXguYXV0aCIsInN1YiI6ImFkbWluQGV4YW1wbGUuY29tIiwiaXNzdWVyX2lkIjoiOTA3MjkzMDMtZDMwZC00YmQ5LTkwMTYtNDljMThjZmY4YjUxIiwidHlwZSI6MH0.G1kjXiGX76BqpytmLdXtjLF9s9K5CVm4ScNMIaKlkwE
```

Now, we can create a new user as follows:
```bash
curl -s -S -i -X POST -H "Content-Type: application/json" http://localhost/users -d '{"email":"user@example.com", "password":"12345678"}'

HTTP/1.1 201 Created
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 08:45:57 GMT
Content-Type: application/json
Content-Length: 0
Connection: keep-alive
Location: /users/f31f8a0a-11b1-4aa6-a4a3-9629378c0326
Strict-Transport-Security: max-age=63072000; includeSubdomains
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: *
Access-Control-Allow-Headers: *
```
You can obtain the user ID via `Location`. The ID of the `user@example.com` is `f31f8a0a-11b1-4aa6-a4a3-9629378c0326`. 

After creating the new user, we have two users on the system as `admin@example.com` and `user@example.com`.
Then, the admin creates multiple Things called `admin-thing-1` and `admin-thing-2`.

```bash
curl -s -S -i -X POST -H "Content-Type: application/json" -H "Authorization: $token" http://localhost/things/bulk -d '[{"name": "a
dmin-thing-1"}, {"name": "admin-thing-2"}]'

HTTP/1.1 201 Created
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 08:53:38 GMT
Content-Type: application/json
Content-Length: 241
Connection: keep-alive
Access-Control-Expose-Headers: Location

{"things":[{"id":"c3d75452-ae00-4aea-84f9-29ab79fd0d26","name":"admin-thing-1","key":"4fb36389-f7a5-424d-8c4f-da5c9e91f3c5"},{"id":"ee589c61-0b98-4176-9da0-d91913087be6","name":"admin-thing-2","key":"410f5889-c756-470d-bd65-2e99b4ecc679"}]}
```
```bash
export th1=c3d75452-ae00-4aea-84f9-29ab79fd0d26
export th2=ee589c61-0b98-4176-9da0-d91913087be6
```

Mainflux identifies `admin@example.com` via the token provided through the `Authorization` request header. On top of that, Mainflux claims ownership of things (`admin-thing-1` and `admin-thing-2`) on the `admin@example.com`. So that, the creator of Things (in this case `admin@example.com`) is going to have `read`, `write` and `delete` policies on the Thing.

If `user@example.com` logs in the system, `user@example.com` cannot access the things created by the `admin@example.com` due to lack of policies.

The next step is creating the user and things Groups respectively. You can create groups as follows:

```bash
curl -s -S -i -X POST -H "Content-Type: application/json" -H "Authorization: $token" http://localhost/groups -d '{"name": "user_group"}'
HTTP/1.1 201 Created
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 09:24:39 GMT
Content-Type: application/json
Content-Length: 0
Connection: keep-alive
Location: /groups/01FHWFFMME9N2N26DG0DMNRWRW
Access-Control-Expose-Headers: Location
```
```bash
curl -s -S -i -X POST -H "Content-Type: application/json" -H "Authorization: $token" http://localhost/groups -d '{"name": "thing_group"}'
HTTP/1.1 201 Created
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 09:24:58 GMT
Content-Type: application/json
Content-Length: 0
Connection: keep-alive
Location: /groups/01FHWFG78DSYA458D8ST4YQ9Y9
Access-Control-Expose-Headers: Location
```

Again, you can obtain group IDs via `Location` in response. It is convenient to store them in variables.
```bash
export ug=01FHWFFMME9N2N26DG0DMNRWRW
export tg=01FHWFG78DSYA458D8ST4YQ9Y9
```

After creating groups, we are ready to assign new members to groups. Let's start with the user group.
```bash
curl -s -S -i -X POST -H "Content-Type: application/json" -H "Authorization: $token" http://localhost/groups/$ug/members -d '{"members":["f31f8a0a-11b1-4aa6-a4a3-9629378c0326"], "type":"users"}'

HTTP/1.1 200 OK
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 09:37:05 GMT
Content-Type: application/json
Content-Length: 0
Connection: keep-alive
Access-Control-Expose-Headers: Location
```
If you remember, `f31f8a0a-11b1-4aa6-a4a3-9629378c0326` is the ID of the `user@example.com`. Since the `$ug` represents the ID of the user group called `user_group`, we indicated the type of the group as `"users"` in the request body.

Now, we can assign Things to the thing group.
```bash
curl -s -S -i -X POST -H "Content-Type: application/json" -H "Authorization: $token" http://localhost/groups/$tg/members -d '{"members":["c3d75452-ae00-4aea-84f9-29ab79fd0d26", "ee589c61-0b98-4176-9da0-d91913087be6"], "type":"things"}'

HTTP/1.1 200 OK
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 09:42:12 GMT
Content-Type: application/json
Content-Length: 0
Connection: keep-alive
Access-Control-Expose-Headers: Location
```
The same logic applies here as well. The IDs of the things that `admin@example.com` created are `c3d75452-ae00-4aea-84f9-29ab79fd0d26` and `ee589c61-0b98-4176-9da0-d91913087be6`. Since the `$tg` represents the ID of the thing group called `thing_group`, we indicated the type of the group as `"things"` in the request body.

Before moving to the third step, let's analyze the current situation. We have two groups, two users, and two things. The first group is the user group and consists of two users, `admin@example.com` (since the admin created the group) and `user@example.com`. The second group is the thing group. It includes two things created by `admin@example.com`.  `user@example.com` still has no access to things created by `admin@example.com`. You can verify it as:
```bash
curl -s -S -i -X GET -H "Authorization: $usertoken" http://localhost/things/$th1
HTTP/1.1 403 Forbidden
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 09:51:45 GMT
Content-Type: application/json
Content-Length: 60
Connection: keep-alive

{"error":"failed to perform authorization over the entity"}
```
```bash
curl -s -S -i -X GET -H "Authorization: $usertoken" http://localhost/things/$th2
HTTP/1.1 403 Forbidden
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 09:51:49 GMT
Content-Type: application/json
Content-Length: 60
Connection: keep-alive

{"error":"failed to perform authorization over the entity"}
```

The `$usertoken` is the token for `user@example.com`. As you can see, requests to access things are denied.

Now, let's assign group access rights.
```bash
curl -s -S -i -X POST http://localhost/groups/$ug/share -d '{"thing_group_id": "01FHWFG78DSYA458D8ST4YQ9Y9"}' -H 'Content-Type: application/json' -H "Authorization: $token"
HTTP/1.1 200 OK
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 09:59:13 GMT
Content-Type: application/json
Content-Length: 3
Connection: keep-alive
Access-Control-Expose-Headers: Location 
```

Now, all the members of the `user_group` have access to things grouped by `thing_group`. Therefore, `user@example.com` has `read`, `write` and `delete` policies on the things within the thing_group. 

Try to access things as `user@example.com`.

```bash
curl -s -S -i -X GET -H "Authorization: $usertoken" http://localhost/things/$th1
HTTP/1.1 200 OK
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 10:02:19 GMT
Content-Type: application/json
Content-Length: 114
Connection: keep-alive
Access-Control-Expose-Headers: Location

{"id":"c3d75452-ae00-4aea-84f9-29ab79fd0d26","name":"admin-thing-1","key":"4fb36389-f7a5-424d-8c4f-da5c9e91f3c5"}
```
```bash
curl -s -S -i -X GET -H "Authorization: $usertoken" http://localhost/things/$th2
HTTP/1.1 200 OK
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 10:02:21 GMT
Content-Type: application/json
Content-Length: 114
Connection: keep-alive
Access-Control-Expose-Headers: Location

{"id":"ee589c61-0b98-4176-9da0-d91913087be6","name":"admin-thing-2","key":"410f5889-c756-470d-bd65-2e99b4ecc679"}
```
Successful!

Let's assume, `admin@example.com` does not want to share things with `user@example.com` anymore. In order to achieve that, `admin@example.com` unassigns `user@example.com` from the `user_group`.
```bash
curl -s -S -i -X DELETE -H "Content-Type: application/json" -H "Authorization: $token" http://localhost/groups/$ug/members -d '{"members":["f31f8a0a-11b1-4aa6-a4a3-9629378c0326"], "type":"users"}'

HTTP/1.1 204 No Content
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 10:08:56 GMT
Content-Type: application/json
Connection: keep-alive
Access-Control-Expose-Headers: Location
```

Now, when `user@example.com` tries to access the things, the request will be denied.

```bash
curl -s -S -i -X GET -H "Authorization: $usertoken" http://localhost/things/$th1
HTTP/1.1 403 Forbidden
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 10:10:26 GMT
Content-Type: application/json
Content-Length: 60
Connection: keep-alive

{"error":"failed to perform authorization over the entity"}
```
```bash
curl -s -S -i -X GET -H "Authorization: $usertoken" http://localhost/things/$th2
HTTP/1.1 403 Forbidden
Server: nginx/1.20.0
Date: Wed, 13 Oct 2021 10:10:28 GMT
Content-Type: application/json
Content-Length: 60
Connection: keep-alive

{"error":"failed to perform authorization over the entity"}
```
