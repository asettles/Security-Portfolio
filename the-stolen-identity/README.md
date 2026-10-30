## The Stolen Identity

An attacker got into the Mad Hat Labs tenant by phishing an employee and stealing their authenticated session. The employee was also an owner of **Mad-Hat-Legacy-Sync-Service**, a legacy application with more access than the employee’s own account.

Using Reader access, I worked through the five stages of the lab to see how the attacker used that ownership to gain application access and leave several ways to get back in.

## Environment  
- **Platform:** Live multi-user Azure training tenant.
- **Services:** Microsoft Entra ID and Microsoft Graph.
- **Tools:** Azure portal and Obsidian.
- **Access:** Reader on directory applications.
## Investigation

### 1. Entry

I opened **Mad-Hat-Legacy-Sync-Service → Branding & properties** and checked the internal notes.
The narrative described how Carl from accounting entered his credentials into a fake login page and completed MFA. The attacker then stole the authenticated session, so they could use access that had already passed the MFA challenge.

Carl was still listed as an owner of the legacy application, thus giving the attacker a way to move from his user account into application management.

<img width="1308" height="795" alt="1" src="https://github.com/user-attachments/assets/c9056d75-c05a-46bc-aa93-f4089c907cb7" />


### 2. Escalation

Under **Certificates & secrets**, I found a client secret that expired on **December 31, 2099**.

In the scenario, the attacker used Carl’s ownership rights to add this credential. They could then authenticate as the application through the **client credentials flow** and use its existing application permissions without relying on Carl’s interactive session.

The expiration date was the obvious concern. The long-lived secret gave the attacker a way to maintain access as the application until the credential expired or was removed, making it a method of persistence.

<img width="1281" height="649" alt="2" src="https://github.com/user-attachments/assets/e1b3cd69-0534-4f22-8a7a-e8651afcceb7" />


### 3. Pivot

The attacker wanted a way to keep access if the original secret was removed or rotated. They created **Mad-Hat-Labs-App** and used the compromised user’s ownership rights to add its service principal to the **Owners** list of **Mad-Hat-Legacy-Sync-Service**.

This created another method of persistence. Instead of depending on one secret, the attacker retained an ownership path that could allow them to add replacement credentials.

I reviewed the legacy application’s **API permissions**, which included **Directory.Read.All** and **User.ReadWrite.All** Microsoft Graph application permissions with admin consent. These permissions explained the potential impact: access to the legacy application could allow the attacker to read directory information and read or modify user profiles across the tenant.

I then checked the legacy application’s **Owners** list and followed the relationship to **Mad-Hat-Labs-App → Branding & properties**, where I reviewed the rogue application’s internal notes.

<img width="1297" height="786" alt="3" src="https://github.com/user-attachments/assets/7319554a-9ba9-440a-8847-ad4d8d233150" />


### 4. Persistence

The attacker wanted another way back in if the secret on the legacy application was removed. Using Carl’s access, they added a custom scope under **Expose an API** on **Mad-Hat-Legacy-Sync-Service**.

This gave the rogue application a permission it could ask users to approve. The attacker could then send a phishing link that prompted users to grant access to the legacy API. This access path would be separate from the original client secret.

I opened **Expose an API**, found the enabled **Legacy.Sync** scope, and reviewed its **User consent display name**. The scope supported the attacker’s backup plan, but access still required consent and the backend accepting the requests.

<img width="1339" height="703" alt="4" src="https://github.com/user-attachments/assets/ea807937-6ff0-4ac8-a3ac-1c0f288bc454" />


### 5. Loot

With the custom scope in place, the attacker’s next step was to get users to approve access for the rogue application. They could send a phishing link that opened a Microsoft consent prompt asking for permission to access the legacy API.

After successful authorization, Microsoft would send an authorization code to the attacker-controlled redirect URI. The rogue application could exchange that code for a token and use it to call the legacy API on the user’s behalf.

This gave the attacker another route besides stealing credentials again. The victim could approve access while already signed in on a corporate device, allowing the rogue app to receive access through the user’s consent. This does not bypass every Conditional Access policy, but it abuses an authorized sign-in rather than requiring the attacker to sign in separately as the user.

This would not automatically give the attacker the legacy application’s Microsoft Graph permissions. However, if the backend failed to check what the user was allowed to do, the attacker could trick it into performing actions with its own higher permissions. This is a **confused deputy** attack: a trusted application uses its access to carry out a request it should have rejected.

The consent grant also needed separate attention during cleanup. Resetting a password, revoking sign-in sessions, or requiring MFA would not remove the grant itself.

I reviewed the suspicious redirect URI and used CyberChef to URL-decode it.
<img width="1765" height="892" alt="5" src="https://github.com/user-attachments/assets/0211f70d-9cc2-4a34-95db-563e124f15b9" />


## Recommendations

1. Remove the rogue service principal from the legacy application’s Owners list and remove any other unauthorized owners.
2. Revoke the attacker-added client secret and review the remaining credentials for appropriate expiration dates.
3. Explicitly revoke unauthorized OAuth consent grants. Password resets and session revocation do not remove the grants themselves.
4. Remove the attacker-added API scope and redirect URI after checking for legitimate dependencies.
5. Confirm that all unauthorized access paths have been removed, including credentials, application ownership, and consent grants.
6. Review and reduce Microsoft Graph application permissions to what the application actually needs.
7. Make sure the backend checks what a user is allowed to do before carrying out requests with its own permissions.
8. Restrict application registration to approved users if general users do not need it.
9. Review application Owners lists alongside privileged directory roles, and alert on new owners, credentials, consent grants, and redirect URIs.

## What I learned

1. A user does not need an admin role to have access that can cause serious damage.
2. Application ownership matters just as much as the permissions assigned to the app.
3. Removing one credential can leave other ways back in.
4. Removing a secret does not remove an owner or a consent grant.
5. Delegated permissions and application permissions are different and should not be treated as interchangeable.
6. Reviewing identity security means following the access relationships, not just looking at a list of privileged users.
