## IAM - Users & Groups
- IAM stands for Identity and Access Management.
- IAM is a global service.
- Root account is created by default. It shouldn't be used or shared.
- Users are people within your organization and can be grouped.
- Groups can only contain users, not other groups.
- Users don't have to belong to a group (not a good practise), and user can belong to multiple groups.

### Permissions -
- Users or Groups can be assigned JSON documents called policies which define the permissions of the users.
- In AWS, you apply the "Least Priviledge Principle" - don't give permissions than a user needs.

> [!TIP]
> Signin URLs are generally `<accountId>.sigin.aws.amazon.com/console`. You can also change the `Account Alias` to a custom text so that the signin URL will become `<account_alias>.sigin.aws.amazon.com/console`. Note that the account alias has to be unique and now you can use both account id or account alias to sign in.




