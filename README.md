# Proxmox VNC Access Integration for VPS Customers

This guide describes how to configure Proxmox VE in order to provide VPS customers with access to the **noVNC console** through a custom web frontend.  
It covers the required Proxmox role setup, API usage for user management, ticket generation, and frontend integration.

---

## 1. Prerequisites

- A working Proxmox VE cluster or standalone node.  
- Administrative access to the Proxmox API.  
- A backend service capable of communicating with the Proxmox API.  
- A frontend application (web-based) where the VNC console will be embedded via iframe.  
- Proper DNS configuration ensuring that the Proxmox node is accessible under your chosen domain (e.g., `node.mydomain.com`).  

---

## 2. Role Setup in Proxmox

Before creating users, define a dedicated role for customer console access:

1. In the **Proxmox Web UI**, navigate to:  
   `Datacenter > Permissions > Roles > Add`

2. Create a role named for example **PanelCustomer**.

3. Assign the following privilege:  
   - `VM.Console`

This ensures that users assigned this role can view and access their VM console but will not have unnecessary privileges.

---

## 3. User Creation via Proxmox API

When provisioning a new VPS for a customer, create a corresponding Proxmox user account.

**Endpoint:**  
```
POST /access/users
```

**Request body example:**
```json
{
  "userid": "customer-vps123@pve",
  "password": "StrongPassword123!",
  "comment": "VPS ID 123 for customer",
  "expire": 0
}
```

- `userid`: Unique identifier for the user (can match VPS ID).  
- `password`: Customer’s login password.  
- `comment`: Optional, e.g., “VPS ID …”.  
- `expire`: `0` means no expiration date.  

---

## 4. Assign Role to User

After the user is created, assign the previously created role to the VM resource.

**Endpoint:**  
```
POST /access/acl
```

**Request body example:**
```json
{
  "path": "/vms/123",
  "users": "customer-vps123@pve",
  "roles": "PanelCustomer"
}
```

- `path`: Resource path (specific VM ID).  
- `users`: Proxmox user ID.  
- `roles`: The role created earlier (`PanelCustomer`).  

This ensures the user only has console access to their own VM.

---

## 5. Generating a Ticket for Authentication

To allow customers to access the noVNC console, generate an authentication ticket from your backend.  

**Endpoint:**  
```
POST /access/ticket
```

**Request body example:**
```json
{
  "username": "customer-vps123@pve",
  "password": "StrongPassword123!"
}
```

**Response (excerpt):**
```json
{
  "data": {
    "ticket": "PVE:abcd1234...",
    "username": "customer-vps123@pve",
    "CSRFPreventionToken": "12345678:xyz..."
  }
}
```

The `ticket` field is the critical value that needs to be passed to the frontend.

---

## 6. Frontend Integration

1. **Store the authentication ticket as a cookie**  
   In your frontend, after receiving the ticket from the backend:

   ```javascript
   setCookie("PVEAuthCookie", response?.ticket, {
       domain: ".mydomain.com",
       path: "/",
       secure: window.location.protocol === "https:",
       sameSite: "none",
       maxAge: 86400000
   });
   ```

   Ensure that your Proxmox node (`node.mydomain.com`) is served under the same domain, so the cookie is valid.

---

2. **Embed the Proxmox noVNC console in an iframe**  

   ```jsx
   <iframe 
     style={{ height: "100%", width: "100%" }}
     src={`https://node.mydomain.com/?console=kvm&vmid=<VMID>&node=<NODE>&resize=scale&novnc=1`}
   />
   ```

   Replace:
   - `<VMID>` with the actual VM ID.  
   - `<NODE>` with the Proxmox node name.  

   The noVNC console will automatically use the `PVEAuthCookie` set earlier to authenticate the session.

---

## 7. Summary

By following the above steps, you have:

1. Created a restricted role (`PanelCustomer`) for console-only access.  
2. Automated user account creation and role assignment via the Proxmox API.  
3. Implemented secure ticket-based authentication.  
4. Integrated Proxmox noVNC console into your frontend with seamless single sign-on.  

This setup enables VPS customers to access their Proxmox VNC console directly from your hosting panel without exposing administrative privileges.