Integrating **SonarQube** with **Azure Active Directory (Azure AD)** allows users to log in using their Azure AD credentials. Here’s how you can set it up:

---

1. Craete a user in azure

![image](https://github.com/user-attachments/assets/b4030912-fd77-4280-b9a9-a8dfa289c2d2)


### **🔹 Prerequisites**
1. **SonarQube**: Ensure SonarQube is up and running (preferably Enterprise Edition or higher for SAML support).  
2. **Azure AD Admin Access**: You need access to the **Azure AD portal** to create an app registration.  
3. **SonarQube Admin Access**: You need admin rights in SonarQube to configure authentication.  

---

### **🔹 Step-by-Step Integration Guide**

#### **1️⃣ Register SonarQube in Azure AD**
1. **Go to Azure Portal** → **Azure Active Directory** → **App Registrations** → **New Registration**  
2. Enter:
   - **Name**: `SonarQube`
   - **Supported account types**: Choose as per your org’s need
   - **Redirect URI**:  
     ```
     https://your-sonarqube-url/oauth2/callback/saml
     ```
3. Click **Register** and note the **Application (Client) ID**.  




---

#### **2️⃣ Configure Authentication in Azure AD**
1. Go to **Authentication** → Click **Add a platform** → Select **Web**.  
2. Set the Redirect URI:
   ```
   https://your-sonarqube-url/oauth2/callback/saml
   ```
3. Enable **ID tokens** and **Save**.  

---

#### **3️⃣ Configure SAML Attributes in Azure AD**
1. **Go to "Enterprise Applications"** → Find **SonarQube** → Click **Single sign-on**  
2. Choose **SAML** → **Basic SAML Configuration**  
3. Configure:
   - **Identifier (Entity ID)**: `https://your-sonarqube-url`
   - **Reply URL** (ACS URL): `https://your-sonarqube-url/oauth2/callback/saml`
   - **Logout URL**: `https://your-sonarqube-url`
4. **Attributes & Claims**:
   - `user.mail` → `NameID`
   - `user.principalname` → `username`
   - `user.givenname` → `firstName`
   - `user.surname` → `lastName`
5. Download the **Federation Metadata XML** (will be used in SonarQube).  

---

#### **4️⃣ Configure SonarQube for SAML**
1. **Log in to SonarQube** as an admin  
2. Go to **Administration** → **Security** → **SAML**  
3. Set:  
   - **Provider Name**: `Azure AD`
   - **SAML Login URL**: *(From Azure AD SSO page)*
   - **SAML Logout URL**: *(From Azure AD SSO page)*
   - **X.509 Certificate**: *(Copy from Azure AD metadata XML)*
   - **Application ID**: *(Client ID from Azure AD)*
   - **Keycloak compatibility mode**: ✅ Enabled  
4. **Save & Enable** SAML authentication.  

---

### **🔹 Testing & Troubleshooting**
1. **Logout** and try **Sign in with Azure AD** in SonarQube.  
2. If login fails, check:
   - SonarQube logs (`sonar.log`)
   - Azure AD **Enterprise Application logs**
   - Ensure **Claims Mapping** is correctly configured  

---

Would you like automation using Terraform or Helm for this setup? 🚀

3. 


