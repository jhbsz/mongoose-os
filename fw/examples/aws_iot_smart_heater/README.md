# AWS IoT button example

This is a draft of the "AWS IoT Smart Heater" example.

## Build instructions

```bash
# Flash the firmware (you might need to adjust the architecture)
mos flash mos-esp8266

# Get device id (e.g. esp8266_DA84C1), we'll need it later
mos config-get device.id

# Put init.js on the device
mos put init.js

# Set wifi configuration
mos config-set wifi.sta.enable=true wifi.ap.enable=false \
               wifi.sta.ssid=WIFI_SSID wifi.sta.pass=WIFI_PASS

# Setup device to connect to AWS IoT
mos aws-iot-setup --aws-iot-policy=mos-default

# Attach console
mos console

# Now, in another terminal, instantiate AWS stack.
# First of all, install node_modules for the helpers
npm --prefix ../helpers/cloudformation-helpers install ../helpers/cloudformation-helpers

# We'll also need to create a separate S3 bucket for helper functions:
aws s3 mb s3://my-cf-helpers

# Now, "package" the template. Packaging includes copying source code of the
# helper functions from local machine to the s3 bucket we created above,
# and adjusting the template appropriately. It's all done in one step:
aws cloudformation package \
    --template-file aws_iot_heater_template.yaml \
    --s3-bucket my-cf-helpers \
    --output-template-file packaged_template.yaml

# The command above has created a new template file: packaged-template.yaml.
# Now, instantiate AWS stack (replace <device_id> with the ID you obtained by
# config-get in the beginning)
#
# In the example command I use stack name "my-heater", but you can
# use any other name.
aws cloudformation create-stack \
    --stack-name my-heater \
    --parameters \
        ParameterKey=DeviceID,ParameterValue=<device_id> \
    --capabilities CAPABILITY_IAM \
    --template-body file://packaged_template.yaml

# Wait until the stack creation is completed (it may take a few minutes).
aws cloudformation wait stack-create-complete --stack-name my-heater

# Alternatively, you can use the web UI to check the status and read event
# details: https://console.aws.amazon.com/cloudformation/home

# When the stack is created, get the name of the created S3 bucket:
aws cloudformation describe-stacks --stack-name my-heater

# look for the following:
#  ...
#  {
#      "Description": "Name of the newly created S3 bucket", 
#      "OutputKey": "S3BucketName", 
#      "OutputValue": "<my-s3bucket-name>"
#  }
#  ...
#
# Copy the actual value of "<my-s3bucket-name>", and use it to put two files
# on the S3 bucket:
aws s3 cp bucket/index.html s3://<my-s3bucket-name> --acl public-read
aws s3 cp bucket/index.js s3://<my-s3bucket-name> --acl public-read

# Download two files of Cognito SDK, and also put them on the S3 bucket:
wget https://raw.githubusercontent.com/aws/amazon-cognito-identity-js/master/dist/aws-cognito-sdk.min.js
wget https://raw.githubusercontent.com/aws/amazon-cognito-identity-js/master/dist/amazon-cognito-identity.min.js
aws s3 cp aws-cognito-sdk.min.js s3://<my-s3bucket-name> --acl public-read
aws s3 cp amazon-cognito-identity.min.js s3://<my-s3bucket-name> --acl public-read

# Now, navigate to the index page of your S3 bucket:
# https://<my-s3bucket-name>.s3.amazonaws.com/index.html
```

You'll see latest graph of the temperature reported from the device, current
heater status (on/off), and the switch. Switching the heater is possible only
for authenticated and authorized users; first of all, you need to sign up:
click the "Sign Up" link at the top of the page, enter credentials, and push
the "Sign Up" button. You'll be prompted for the verification code sent to your
email, and after the code is entered, you'll be signed in.

If you try to switch the heater status, you'll get the message saying that
you are not authorized to do that. Now, you need to authorize your user to
manage heater.

For that, navigate to the [AWS Cognito console](https://console.aws.amazon.com/cognito/home), click "Manage your User
Pools", select "user_pool_for_<device_id>", click "Users and groups", and
you'll see the list of your registered users. At this point, you should see
your user there; click on it, then click "Add to group", select the group
"heater_admins", and finally click "Add to group". After that, you can sign out
from your heater app, sign in back, and switching the heater should result
in the state being changed.
