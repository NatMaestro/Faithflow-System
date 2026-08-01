# 🔌 Frontend ↔️ Backend Integration Guide

## ✅ How Multi-Tenancy Works in Your App

### The Magic: Subdomain Auto-Detection

**User Journey:**

1. **User visits:** `https://testchurch.faithflows.com`
2. **Frontend detects:** Subdomain = "testchurch"
3. **Frontend calls:** `http://testchurch.localhost:8000/api/v1/members/`
4. **Backend returns:** ONLY Test Church members
5. **NO CODE CHANGES NEEDED!** Same code works for all churches!

---

## 🚀 Using the API Client

### Step 1: Import the API Client

```typescript
import { api } from "@/utils/api";
```

### Step 2: Use It Anywhere

```typescript
// Example: Login Component
async function handleLogin(email: string, password: string) {
  try {
    const response = await api.login(email, password);
    // Token is automatically stored!
    console.log("Logged in:", response.user);
  } catch (error) {
    console.error("Login failed:", error);
  }
}

// Example: Members List Component
async function loadMembers() {
  try {
    const members = await api.getMembers();
    // Returns ONLY current church's members!
    setMembers(members.results);
  } catch (error) {
    console.error("Failed to load members:", error);
  }
}

// Example: Create Payment
async function createPayment(paymentData) {
  try {
    const payment = await api.createPayment(paymentData);
    console.log("Payment created:", payment);
  } catch (error) {
    console.error("Payment failed:", error);
  }
}
```

---

## 🎯 Real Component Example

### Before (Using local db.json):

```typescript
// OLD WAY - Frontend manages data
import { db } from "@/lib/db";

function MembersPage() {
  const [members, setMembers] = useState([]);

  useEffect(() => {
    // Load from local db.json
    const churchMembers = db.members.filter(
      (m) => m.churchId === currentChurch.id
    );
    setMembers(churchMembers);
  }, []);

  return <MembersList members={members} />;
}
```

### After (Using Backend API):

```typescript
// NEW WAY - Backend manages data
import { api } from "@/utils/api";

function MembersPage() {
  const [members, setMembers] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    async function loadMembers() {
      try {
        setLoading(true);
        // API automatically knows which church based on subdomain!
        const data = await api.getMembers();
        setMembers(data.results);
      } catch (error) {
        console.error("Error loading members:", error);
      } finally {
        setLoading(false);
      }
    }

    loadMembers();
  }, []);

  return loading ? <Loader /> : <MembersList members={members} />;
}
```

---

## 🌐 How Subdomain Detection Works

### Development Mode:

**User visits:**

```
http://testchurch.localhost:5173  ← Your frontend dev server
```

**Frontend JavaScript:**

```typescript
const subdomain = getCurrentSubdomain();
// Returns: "testchurch"

const apiURL = getAPIBaseURL();
// Returns: "http://testchurch.localhost:8000/api/v1"
```

**API calls automatically go to:**

```
http://testchurch.localhost:8000/api/v1/members/
         ^^^^^^^^^^                  ^^^^^^^^
         Same subdomain!             Backend API
```

---

### Production Mode:

**User visits:**

```
https://olamchurch.faithflows.com  ← Your deployed frontend
```

**Frontend JavaScript:**

```typescript
const subdomain = getCurrentSubdomain();
// Returns: "olamchurch"

const apiURL = getAPIBaseURL();
// Returns: "https://olamchurch.faithflows.com/api/v1"
```

**API calls automatically go to:**

```
https://olamchurch.faithflows.com/api/v1/members/
         ^^^^^^^^^^
         Same subdomain = Olam Church data!
```

---

## 🔐 Login Flow

### When User Logs In:

```typescript
// 1. User is on: http://testchurch.localhost:5173
// 2. Subdomain detected: "testchurch"
// 3. Login calls: http://testchurch.localhost:8000/api/v1/auth/login/

await api.login("admin@testchurch.com", "admin123");

// 4. Backend checks: "Is this user in Test Church database?"
//    YES → Returns JWT token
//    NO → Returns error

// 5. Token stored in localStorage
// 6. All future API calls include this token
// 7. All API calls automatically go to testchurch subdomain
```

---

## 📊 Complete Example: Members Management

```typescript
import { api } from "@/utils/api";

// Component automatically works for ANY church!
// No need to know which church - subdomain handles it!

export function MembersManager() {
  const [members, setMembers] = useState([]);

  // Load members (automatically from current church)
  const loadMembers = async () => {
    const data = await api.getMembers();
    setMembers(data.results);
  };

  // Create member (automatically in current church)
  const createMember = async (memberData) => {
    const newMember = await api.createMember(memberData);
    setMembers([...members, newMember]);
  };

  // Update member (automatically in current church)
  const updateMember = async (id, memberData) => {
    const updated = await api.updateMember(id, memberData);
    setMembers(members.map((m) => (m.id === id ? updated : m)));
  };

  // Delete member (automatically in current church)
  const deleteMember = async (id) => {
    await api.deleteMember(id);
    setMembers(members.filter((m) => m.id !== id));
  };

  return <div>{/* Your UI here - same code for all churches! */}</div>;
}
```

---

## 🎯 Key Points:

### ✅ What You DON'T Do:

- ❌ Don't hardcode URLs for each church
- ❌ Don't manually switch between churches
- ❌ Don't check which church in your components

### ✅ What You DO:

- ✅ Use `api.getMembers()` - automatically gets current church
- ✅ Use `api.createEvent()` - automatically creates in current church
- ✅ Use `api.getPayments()` - automatically gets current church's payments

**The subdomain in the URL determines everything!**

---

## 🌍 Deployment Architecture:

```
Frontend (Vercel/Netlify):
├── testchurch.faithflows.com  → Your React app (same code)
├── olamchurch.faithflows.com  → Your React app (same code)
└── deeperlife.faithflows.com  → Your React app (same code)
         ↓ All call ↓
Backend (Railway/Render):
├── testchurch schema → Test Church data
├── olamchurch schema → Olam Church data
└── deeperlife schema → Deeper Life data
```

**ONE frontend deployment serves ALL churches!**  
**ONE backend deployment serves ALL churches!**  
**Data isolated by subdomain!**

---

## 💡 Summary:

**Your Concern:** "Do I change base URL for each church?"  
**Answer:** ❌ **NO!** The subdomain in the browser URL **automatically** determines the API URL.

**How:**

1. User visits: `testchurch.localhost:5173`
2. Frontend detects: `subdomain = "testchurch"`
3. Frontend calls: `testchurch.localhost:8000/api/v1/...`
4. **All automatic!** No manual switching!

**Same frontend code works for 1,000 churches!** 🎉

---

**Does this clarify how it works?** The subdomain is the "key" that unlocks which church's data to use - both in frontend and backend!
