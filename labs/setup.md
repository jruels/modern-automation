# Setup Student VM's

# Configure AWS credentials

Your instructor has provided you with a set of AWS credentials for this class. You do not need to create your own access keys.

The credentials include:

* A **console sign-in URL**, **username**, and **password**
* An **Access key ID** and **Secret access key**
* A **Region** (`us-west-1`)

---

### **Step 1: Log into the AWS Console**

1. In a browser, open the **console sign-in URL** provided by your instructor.
2. Sign in with the **username** and **password** provided by your instructor.

---

### **Step 2: Use AWS Configure**

1. Run `aws configure` in the Visual Studio Code terminal.
2. Supply the required information.
   * **Access key ID** and **Secret access key** provided by your instructor
   * Region = `us-west-1`
3. Select the default option for remaining.

---

### **Step 3: Test AWS CLI**

1. Confirm that `aws` can use the credentials.

   ```
   aws sts get-caller-identity
   ```

2. You should see your account information returned.



## Congratulations

Congratulations, you've successfully configured your machine.
