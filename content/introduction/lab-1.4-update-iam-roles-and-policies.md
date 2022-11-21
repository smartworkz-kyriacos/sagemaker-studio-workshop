+++
chapter = false
title = "Lab 1.4 Update IAM Roles and Policies"
weight = 10

+++
**Update IAM Roles and Policies**

In \[ \]:

    import boto3
    import sagemaker
    import time
    from time import gmtime, strftime
    
    sagemaker_session = sagemaker.Session()
    role = sagemaker.get_execution_role()
    bucket = sagemaker_session.default_bucket()
    region = boto3.Session().region_name
    
    from botocore.config import Config
    
    config = Config(retries={"max_attempts": 10, "mode": "adaptive"})
    
    iam = boto3.client("iam", config=config)
    

**Get SageMaker Execution Role Name**

In \[ \]:

    role_name = role.split("/")[-1]
    
    print("Role name: {}".format(role_name))
    

In \[ \]:

    setup_iam_roles_passed = False
    

**Pre-Requisite: SageMaker notebook instance ExecutionRole contains `AdministratorAccess` Policy.**

> _Note: The permissions used here are for demonstration purposes only. Please follow the least-privilege security principles appropriate for your environment._

In \[ \]:

    admin = False
    
    post_policies = iam.list_attached_role_policies(RoleName=role_name)["AttachedPolicies"]
    for post_policy in post_policies:
        if post_policy["PolicyName"] == "AdministratorAccess":
            admin = True
            setup_iam_roles_passed = True
            print("[OK] You are all set up to continue with this workshop!")
            break
    
    if not admin:   
            print("*************** [ERROR] SageMakerExecutionRole needs the AdministratorAccess policy attached. *****************")
    

**If you see an ERROR message ^^ above ^^, please attach the AdministratorAccess Policy to the SageMaker notebook instance ExecutionRole.**

_Note: The permissions used here are for demonstration purposes only. Please follow the least-privilege security principles appropriate for your environment._

In \[ \]:

    # if not admin:
    #     pre_policies = iam.list_attached_role_policies(RoleName=role_name)["AttachedPolicies"]
    
    #     required_policies = ["IAMFullAccess"]
    
    #     for pre_policy in pre_policies:
    #         for role_req in required_policies:
    #             if pre_policy["PolicyName"] == role_req:
    #                 print("Attached: {}".format(pre_policy["PolicyName"]))
    #                 try:
    #                     required_policies.remove(pre_policy["PolicyName"])
    #                 except:
    #                     pass
    
    #     if len(required_policies) > 0:
    #         print(
    #             "*************** [ERROR] You need to attach the following policies in order to continue with this workshop *****************\n"
    #         )
    #         for required_policy in required_policies:
    #             print("Not Attached: {}".format(required_policy))
    #     else:
    #         print("[OK] You are all set to continue with this notebook!")
    # else:
    #     print("[OK] You are all set to continue with this notebook!")
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "AdministratorAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "AmazonSageMakerFullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "IAMFullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "AmazonS3FullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "ComprehendFullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "AmazonAthenaFullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "SecretsManagerReadWrite"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "AmazonRedshiftFullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "AmazonEC2ContainerRegistryFullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "AWSStepFunctionsFullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "AmazonKinesisFullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "AmazonKinesisFirehoseFullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

In \[ \]:

    # from botocore.exceptions import ClientError
    
    # try:
    #     policy = "AmazonKinesisAnalyticsFullAccess"
    #     response = iam.attach_role_policy(PolicyArn="arn:aws:iam::aws:policy/{}".format(policy), RoleName=role_name)
    #     print("Policy {} has been succesfully attached to role: {}".format(policy, role_name))
    # except ClientError as e:
    #     if e.response["Error"]["Code"] == "EntityAlreadyExists":
    #         print("[OK] Policy is already attached.")
    #     elif e.response["Error"]["Code"] == "LimitExceeded":
    #         print("[OK]")
    #     else:
    #         print("*************** [ERROR] {} *****************".format(e))
    
    # time.sleep(5)
    

**_Final Check_**

In \[ \]:

    # role = iam.get_role(RoleName=role_name)
    post_policies = iam.list_attached_role_policies(RoleName=role_name)["AttachedPolicies"]
    
    required_policies = [
        "AdministratorAccess",
    #     "SecretsManagerReadWrite",
    #     "IAMFullAccess",
    #     "AmazonS3FullAccess",
    #     "AmazonAthenaFullAccess",
    #     "ComprehendFullAccess",
    #     "AmazonEC2ContainerRegistryFullAccess",
    #     "AmazonRedshiftFullAccess",
    #     "AWSStepFunctionsFullAccess",
    #     "AmazonSageMakerFullAccess",
    #     "AmazonKinesisFullAccess",
    #     "AmazonKinesisFirehoseFullAccess",
    #     "AmazonKinesisAnalyticsFullAccess",
    ]
    
    admin = False
    
    for post_policy in post_policies:
        if post_policy["PolicyName"] == "AdministratorAccess":
            admin = True
            try:
                required_policies.remove(post_policy["PolicyName"])
            except:
                break
        else:
            try:
                required_policies.remove(post_policy["PolicyName"])
            except:
                pass
    
    if not admin and len(required_policies) > 0:
        print("*************** [ERROR] RE-RUN THIS NOTEBOOK *****************")
        for required_policy in required_policies:
            print("Not Attached: {}".format(required_policy))
    else:
        setup_iam_roles_passed = True
        print("[OK] You are all set up to continue with this workshop!")
    

In \[ \]:

    %store setup_iam_roles_passed
    

In \[ \]:

    %store
    

**Release Resources**

In \[ \]:

    %%html
    
    <p><b>Shutting down your kernel for this notebook to release resources.b>p>
    <button class="sm-command-button" data-commandlinker-command="kernelmenu:shutdown" style="display:none;">Shutdown Kernelbutton>
            
    <script>
    try {
        els = document.getElementsByClassName("sm-command-button");
        els[0].click();
    }
    catch(err) {
        // NoOp
    }    
    script>
    

In \[ \]:

    %%javascript
    
    try {
        Jupyter.notebook.save_checkpoint();
        Jupyter.notebook.session.delete();
    }
    catch(err) {
        // NoOp
    }