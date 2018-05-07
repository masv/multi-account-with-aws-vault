footer: Martin Svangren — Forza Football
slidenumbers: true

# Secure multi-account AWS with aws-vault

---

## Why multi-account?

* Separate concerns (by product, business unit, etc.)
* Consolidated billing (specified by account)
* Security

---

## The bastion/security/identity account

* AWS account with no compute resources, only IAM.
* A single set of IAM credentials per user.
* Users *assume roles* in other accounts.

---

## Examples

Group *Developers* can access all resources in the *Product* account.

Group *IT* can access all resources in the *Corporate* account and IAM in the *Bastion* account.

Group *Finance* can access billing details in the *Master* account.

---

## This is useful even if you only have a Master account and a Bastion account

---

* Manage *users* in the Bastion account.
* Manage *roles* in your Master account.

---

## Setting up

---

In your *Master* account, create a new IAM role. Make the trusted entity your Bastion account.

![inline](create-role-1.jpeg)

---

Attach permissions.

![inline](create-role-2.jpeg)

---

Name and create the role.

![inline](create-role-3.jpeg)

---

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBastionAccountUserWithMFA",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<BASTION-ACCOUNT-ID>:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

---

In your *Identity* account, create a policy that allows users to assume the role in the *Master* account.

![inline](create-policy-1.jpeg)

---

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::<MASTER-ACCOUNT-ID>:role/DeveloperRole"
        }
    ]
}
```

---

Create a *Developers* group.

![inline](create-group-1.jpeg)

---

And attach the policy.

![inline](create-group-2.jpeg)

---

Create a user.

![inline](create-user-1.jpeg)

---

And add them to the Developers group.

![inline](create-user-2.jpeg)

---

Remember this?

```json
{
  "Condition": {
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"
    }
  }
}
```

The user must activate an MFA device[^1] on their account before they can switch roles.

[^1]: [IAM User Guide: Allow IAM Users to Self-Manage an MFA Device](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_examples_iam_mfa-selfmanage.html)

---

Once signed in to the Identity account, the user can *switch role* and use the AWS console as usual.

![inline](switch-role.jpeg)

---

## CLI setup 🤓

---

~/.aws/config

```
[profile identity]
mfa_serial = arn:aws:iam::<BASTION-ACCOUNT-ID>:mfa/elmerfudd

[profile master]
source_profile = identity
mfa_serial = arn:aws:iam::<BASTION-ACCOUNT-ID>:mfa/elmerfudd
role_arn = arn:aws:iam::<MASTER-ACCOUNT-ID>:role/DeveloperRole
```

---

~/.aws/config

``` [.highlight: 5]
[profile identity]
mfa_serial = arn:aws:iam::<BASTION-ACCOUNT-ID>:mfa/elmerfudd

[profile master]
source_profile = identity
mfa_serial = arn:aws:iam::<BASTION-ACCOUNT-ID>:mfa/elmerfudd
role_arn = arn:aws:iam::<MASTER-ACCOUNT-ID>:role/DeveloperRole
```

---

~/.aws/credentials

```
[identity]
aws_access_key_id = AKIAI6ZCA7DBKEXAMPLE
aws_secret_access_key = A94L...

# No credentials for the master profile,
# it inherits from identity.
```

---

```sh
$ AWS_PROFILE=identity aws sts get-caller-identity
{
    "UserId": "AIDAIMBDIHHP7OEXAMPLE",
    "Account": "111122223333",
    "Arn": "arn:aws:iam::111122223333:user/elmerfudd"
}
```

---

```sh
$ AWS_PROFILE=master aws sts get-caller-identity
Enter MFA code for arn:aws:iam::111122223333:mfa/elmerfudd: 123456
{
    "UserId": "AROAI4MZALGSVCEXAMPLE:botocore-session-1525696159",
    "Account": "222233334444",
    "Arn": "arn:aws:sts::222233334444:assumed-role/DeveloperRole/botocore-session-1525696159"
}
```

---

## Cool, but we still have credentials in clear text on disk.

Enter [aws-vault](https://github.com/99designs/aws-vault).

> AWS Vault stores IAM credentials in your operating system's secure keystore and then generates temporary credentials from those to expose to your shell and applications.

---

Add credentials for the *identity* profile to aws-vault.

```sh
$ aws-vault add identity
Enter Access Key ID: AKIAI6ZCA7DBKEXAMPLE
Enter Secret Access Key: A94L...
Added credentials to profile "identity" in vault
```

---
And remove them from ~/.aws/credentials.

```diff
--- a/credentials
+++ b/credentials
@@ -1,3 +0,0 @@
-[identity]
-aws_access_key_id = AKIAI6ZCA7DBKEXAMPLE
-aws_secret_access_key = A94L...
```

---

Use `aws-vault exec` to generate temporary credentials.

```sh
$ aws-vault exec identity -- aws sts get-caller-identity
Enter token for arn:aws:iam::111122223333:mfa/elmerfudd: 123456
{
    "UserId": "AIDAIMBDIHHP7OEXAMPLE",
    "Account": "111122223333",
    "Arn": "arn:aws:iam::111122223333:user/elmerfudd"
}
```

---

Use `aws-vault exec` to generate temporary credentials.

```sh
$ aws-vault exec master -- aws sts get-caller-identity
Enter token for arn:aws:iam::111122223333:mfa/elmerfudd: 123456
{
    "UserId": "AROAI4MZALGSVCEXAMPLE:1525697128262099000",
    "Account": "222233334444",
    "Arn": "arn:aws:sts::222233334444:assumed-role/DeveloperRole/1525697128262099000"
}
```

---

Use `aws-vault rotate` to rotate your credentials.

```sh
$ aws-vault rotate identity
Rotating credentials for profile "identity" (takes 10-20 seconds)
Done!
```

---

The credentials you had in clear text on disk have now been destroyed and new credentials have been stored in your OS keychain.

---

*Not even you know your credentials now.*

---

## What else can aws-vault do?

---
List your profiles and their credentials sources with `aws-vault list`

```sh
$ aws-vault list
Profile                  Credentials              Sessions
=======                  ===========              ========
identity                 identity                 1525711415
master                   identity                 1525711528
```

---

Run a local EC2 instance metadata server on your machine with `aws-vault server`.

Useful if your are developing apps that use an AWS SDK that reads credentials from instance metadata.

I could not get this to work :disappointed:

---

Last but not least, `aws-vault login`.

```sh
$ aws-vault login master
Enter token for arn:aws:iam::111122223333:mfa/elmerfudd: 123456
```

This opens the AWS console in your web browser, signed in with temporary credentials and the correct role assumed.

It's *magic*.

---

## Takeaways

* Multiple AWS accounts can help you separate concerns and make your AWS bills a little clearer.
* A bastion AWS account can give you an extra layer of security and make it easier to manage users across many accounts.
* aws-vault makes it easier and safer to use the AWS CLI with one or many AWS accounts.

---

## Questions?

Slides: https://github.com/masv/multi-account-with-aws-vault

E-mail: martin@forzafootball.com
