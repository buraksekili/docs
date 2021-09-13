# Authorization

## Policies

Mainflux uses policies to control permissions on entities: **users**, **things**, and **groups**. Under the hood, Mainflux uses [ORY Keto](https://www.ory.sh/keto/) that is the first and only open-source implementation of ["Zanzibar: Google's Consistent, Global Authorization System"](https://www.usenix.org/conference/atc19/presentation/pang)

Policies define permissions for the entities. For example, *which user* has *access* to *a specific thing*? Such policies have three main components: **subject**, **object**, and **relation**.

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


### Things service related policies

- There are 3 policies regarding `Things`: `read`, `write` and `delete`.
- When a user creates a thing, the user will have `read`, `write` and `delete` policies on the `Thing`.
- In order to view a thing, you need `read` policy on that thing.
- In order to update and share the thing, you need a `write` policy on that thing.
- In order to remove a thing, you need a `delete` policy on that thing.


## Example Usage

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

After identifying the requester as `user2`, the Policy service checks that `user2 is allowed to view the "user1-thing"`? Since `user2` has no such policy, the Policy service denies this request.

Now, `user1` wants to share the `"user1-thing"` with `user2`. `user1` can achieve this via HTTP endpoint for sharing things as follows:

```bash
curl -isSX POST http://localhost/things/a1109d52-6281-410e-93ae-38ba7daa9381/share -d '{"user_id":"user_id_2", "policies": ["read", "delete"]}' -H "Authorization: <user1_auth_token>" -H 'Content-Type: application/json'

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

As we expected, the operation is successfully done. The policy server checked that `is user2 allowed to view "user1-thing"`? Since `user2` has a `read` policy on `"user1-thing"`, the Policy server allows this request. 