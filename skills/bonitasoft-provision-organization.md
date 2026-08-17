---
name: Provision a Bonita organization member and grant them work
description: Create a user, place them in the group and role structure, give them a profile so they can call the API at all, and connect them to a process actor so tasks reach them.
api: openapi/bonitasoft-bonita-openapi.yml
operations: [createUser, searchUsers, getUserById, updateUserById, createGroup, searchGroups, createRole, searchRoles, createMembership, searchMemberships, searchProfiles, createProfileMember, searchActors, searchActorMembers, createCustomUserDefinition, searchCustomUsers]
---

# Provision a Bonita organization member

Getting a person into Bonita is four separate things, and skipping any one of
them produces a user who exists but cannot do anything.

1. the **user** record
2. their **membership** (group x role)
3. their **profile** — this is what grants API permissions
4. their connection to a process **actor** — this is what routes tasks to them

Authenticate as an administrator first. All writes need the
`X-Bonita-API-Token` header.

## Steps

1. **Create the structure if it does not exist.**
   - `searchGroups` / `createGroup` — `GET|POST /API/identity/group`. Groups form
     a tree via `parent_group_id`.
   - `searchRoles` / `createRole` — `GET|POST /API/identity/role`.

2. **Create the user.** `createUser` — `POST /API/identity/user`. Set
   `manager_id` if they report to an existing user. Verify with `getUserById` —
   `GET /API/identity/user/{id}`.
   - For a lightweight list (user pickers), `searchUserSummaries` —
     `GET /API/identity/userSummary` — returns id, user name, first name, last
     name and job title without the full payload (Bonita 2026.2+).

3. **Grant the membership.** `createMembership` —
   `POST /API/identity/membership` with `user_id`, `group_id` and `role_id`.
   Membership is a three-way join; the API records who granted it in
   `assigned_by_user_id`. Confirm with `searchMemberships` —
   `GET /API/identity/membership?p=0&c=20&f=user_id%3D<userId>`.

4. **Give them a profile.** `searchProfiles` — `GET /API/portal/profile` — then
   `createProfileMember` — `POST /API/portal/profileMember` with `profile_id`
   plus exactly one of `user_id`, `group_id` or `role_id`.
   **This is the step people miss.** Bonita authorization is profile-based, not
   scope-based: a user with no profile gets 403 on essentially every endpoint.
   Prefer granting to a group or role over an individual user so the grant
   survives staff changes.
   - `createProfile` and `updateProfileById` are **deprecated** in the contract.
     Create profiles through the administration application, not the API; use the
     API for membership only.

5. **Connect them to the work.** `searchActors` —
   `GET /API/bpm/actor?p=0&c=20&f=process_id%3D<processDefinitionId>` — then
   `searchActorMembers` — `GET /API/bpm/actorMember?p=0&c=20&f=actor_id%3D<actorId>`
   — to see who a process's actors resolve to. An actor member is polymorphic:
   exactly one of `user_id`, `group_id`, `role_id` (or a group + role pair) is
   populated. Tasks become pending for whoever the actor resolves to.
   - Actor **mapping** is a design-time concern configured with the process
     deployment; `updateActorById` is deprecated. Read it here, change it in the
     project.

6. **Attach extra attributes if needed.** `createCustomUserDefinition` —
   `POST /API/customuserinfo/definition` — defines a custom field;
   `searchCustomUsers` — `GET /API/customuserinfo/user` — reads values. This is
   Bonita's extensibility point instead of a generic metadata bag, and custom user
   info can drive actor filtering.

## Rules

- **Never delete to "fix" a mistake.** `deleteUserById`, `deleteGroupById` and
  `deleteRoleById` cascade through memberships and actor mappings and can orphan
  running cases. Deactivate or reassign instead.
- **Grant profiles to groups and roles, not people.** Individual grants are how
  permission sprawl starts, and there is no scope surface to audit against.
- **Delegation is not a substitute for provisioning.** From Bonita 2026.2
  (subscription editions) `createDelegationRule` — `POST /API/delegation/rule` —
  lets an absent user's tasks be worked by a delegate without reassignment. Note
  the documented limitation: BDM access is **not** granted automatically, so a
  delegate may still fail to open a task form.
- **`p` and `c` are required on every search.** Omitting them returns 400.
- **No idempotency.** Retrying `createUser` after a timeout may create a second
  user. Search by user name first.

See `authentication/bonitasoft-authentication.yml`,
`data-model/bonitasoft-data-model.yml`, `lifecycle/bonitasoft-lifecycle.yml`.
