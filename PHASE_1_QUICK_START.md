# 🚀 PHASE 1 QUICK START GUIDE
## Fix Critical Bugs - Get Candidate Pages Working

**Timeline:** 2-3 Days  
**Priority:** CRITICAL  
**Status:** Ready to Start

---

## 🎯 OBJECTIVE

Fix the missing API function exports so that candidate pages load without errors. This is a blocking issue preventing the app from building properly.

---

## 🔧 IMPLEMENTATION STEPS

### Step 1: Fix `lib/api.ts` Exports (30 minutes)

**Problem:** Components are importing `fetchCandidates`, `fetchGovernorates`, and `fetchStats` but these functions don't exist.

**Solution:** Add wrapper functions that export the existing API methods with the expected names.

**File to Edit:** `lib/api.ts`

**Add this code at the bottom of the file:**

```typescript
// ============================================
// WRAPPER EXPORTS FOR COMPONENT COMPATIBILITY
// ============================================

/**
 * Fetches candidates with pagination and filters
 * Wrapper for candidatesAPI.getCandidates with data transformation
 */
export async function fetchCandidates(params: {
  page?: number;
  limit?: number;
  query?: string;
  governorate?: string;
  gender?: 'male' | 'female';
} = {}) {
  try {
    const response = await candidatesAPI.getCandidates({
      ...params,
      search: params.query, // Map query to search
      province: params.governorate // Map governorate to province
    });
    
    // Transform response to match component expectations
    return {
      data: Array.isArray(response) ? response : response.data || [],
      total: response.total || response.length || 0,
      page: params.page || 1,
      limit: params.limit || 12
    };
  } catch (error) {
    console.error('Error in fetchCandidates:', error);
    return {
      data: [],
      total: 0,
      page: 1,
      limit: 12
    };
  }
}

/**
 * Fetches governorates list
 * Wrapper for candidatesAPI.getProvinces (naming difference)
 */
export async function fetchGovernorates() {
  try {
    const provinces = await candidatesAPI.getProvinces();
    
    // Transform to expected format
    return Array.isArray(provinces) ? provinces : [];
  } catch (error) {
    console.error('Error in fetchGovernorates:', error);
    return [];
  }
}

/**
 * Fetches election statistics
 * Wrapper for candidatesAPI.getStats
 */
export async function fetchStats(): Promise<Stats> {
  try {
    const stats = await candidatesAPI.getStats();
    return stats;
  } catch (error) {
    console.error('Error in fetchStats:', error);
    // Return empty stats structure on error
    return {
      totalCandidates: 0,
      byProvince: {},
      byParty: {},
      byGender: {},
      byEducation: {}
    };
  }
}

/**
 * Likes a post (placeholder for social features)
 * TODO: Implement when backend is ready
 */
export async function likePost(postId: string): Promise<void> {
  console.warn('likePost called but backend not implemented yet. Post ID:', postId);
  // Simulate API delay
  await new Promise(resolve => setTimeout(resolve, 300));
  // For now, just log - actual implementation will come in Phase 3
}

/**
 * Adds a comment to a post (placeholder for social features)
 * TODO: Implement when backend is ready
 */
export async function addComment(postId: string, content: string): Promise<void> {
  console.warn('addComment called but backend not implemented yet', { postId, content });
  // Simulate API delay
  await new Promise(resolve => setTimeout(resolve, 300));
  // For now, just log - actual implementation will come in Phase 3
}
```

---

### Step 2: Test the Build (5 minutes)

Run the development server to verify no import errors:

```bash
npm run dev
```

**Expected Output:**
```
✓ Ready in 3.2s
○ Local:        http://localhost:3000
```

**Visit these pages to verify:**
- http://localhost:3000/en (home page)
- http://localhost:3000/en/candidates (candidate listing)
- http://localhost:3000/en/governorates (governorate listing)
- http://localhost:3000/en/stats (statistics page)

---

### Step 3: Test Railway API Connection (15 minutes)

Verify the Railway backend is responding properly.

**Create a test script:** `scripts/test-api.js`

```javascript
const axios = require('axios');

const API_URL = 'https://hamlet-unified-complete-2027-production.up.railway.app';

async function testEndpoints() {
  const endpoints = [
    '/health',
    '/api/candidates',
    '/api/governorates',
    '/api/provinces',
    '/api/stats',
    '/api/parties'
  ];
  
  console.log('🧪 Testing Railway API Endpoints\n');
  console.log(`Base URL: ${API_URL}\n`);
  
  for (const endpoint of endpoints) {
    try {
      const response = await axios.get(`${API_URL}${endpoint}`, {
        timeout: 5000
      });
      
      console.log(`✅ ${endpoint}`);
      console.log(`   Status: ${response.status}`);
      console.log(`   Data: ${JSON.stringify(response.data).substring(0, 100)}...`);
      console.log('');
    } catch (error) {
      console.log(`❌ ${endpoint}`);
      console.log(`   Error: ${error.message}`);
      if (error.response) {
        console.log(`   Status: ${error.response.status}`);
        console.log(`   Data: ${JSON.stringify(error.response.data)}`);
      }
      console.log('');
    }
  }
}

testEndpoints();
```

**Run the test:**
```bash
node scripts/test-api.js
```

**Document the results:**
- Which endpoints work? ✅
- Which endpoints return errors? ❌
- What error messages do you see?

---

### Step 4: Handle Backend Issues (30-60 minutes)

**If Railway API is returning errors:**

**Option A: Use Mock Data (Quick Fix)**

Update `lib/api.ts` to use fallback mock data:

```typescript
export async function fetchCandidates(params = {}) {
  try {
    const response = await candidatesAPI.getCandidates(params);
    return {
      data: response.data || response,
      total: response.total || 0,
      page: params.page || 1,
      limit: params.limit || 12
    };
  } catch (error) {
    console.error('API Error - Using mock data:', error);
    
    // FALLBACK: Return mock data
    return {
      data: generateMockCandidates(params.limit || 12),
      total: 100,
      page: params.page || 1,
      limit: params.limit || 12
    };
  }
}

function generateMockCandidates(count: number): Candidate[] {
  return Array.from({ length: count }, (_, i) => ({
    id: i + 1,
    name: `Candidate ${i + 1}`,
    nameArabic: `مرشح ${i + 1}`,
    nameKurdish: `بەربژێر ${i + 1}`,
    age: 35 + (i % 30),
    gender: i % 3 === 0 ? 'female' : 'male',
    education: 'University Degree',
    occupation: 'Public Servant',
    party: ['Independent', 'Progressive Party', 'National Alliance'][i % 3],
    province: ['Baghdad', 'Basra', 'Erbil', 'Mosul'][i % 4],
    constituency: 'District ' + ((i % 10) + 1),
    photo: `https://i.pravatar.cc/150?u=candidate-${i}`
  }));
}
```

**Option B: Fix Railway Backend (Longer Fix)**

If Railway API exists but has issues:

1. **Check Railway logs:**
   ```bash
   # Go to: https://railway.app/dashboard
   # Select your project → View Logs
   ```

2. **Common fixes:**
   - Database not connected (check DATABASE_URL env var)
   - CORS issues (add your domain to CORS whitelist)
   - Server crashed (restart the service)
   - Wrong route paths (check Express router setup)

3. **Deploy updated backend:**
   ```bash
   railway up
   ```

**Option C: Create Simple Backend (2-3 hours)**

If no backend exists, create a minimal Express API:

**File:** `backend-simple/index.js`

```javascript
const express = require('express');
const cors = require('cors');
const app = express();

app.use(cors());
app.use(express.json());

// Mock data
const candidates = require('./mock-candidates.json');
const governorates = require('./mock-governorates.json');

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date() });
});

// Candidates with pagination
app.get('/api/candidates', (req, res) => {
  const { page = 1, limit = 12, search, province, gender } = req.query;
  
  let filtered = candidates;
  
  if (search) {
    filtered = filtered.filter(c => 
      c.name.toLowerCase().includes(search.toLowerCase())
    );
  }
  
  if (province) {
    filtered = filtered.filter(c => c.province === province);
  }
  
  if (gender) {
    filtered = filtered.filter(c => c.gender === gender);
  }
  
  const start = (page - 1) * limit;
  const end = start + parseInt(limit);
  
  res.json({
    data: filtered.slice(start, end),
    total: filtered.length,
    page: parseInt(page),
    limit: parseInt(limit)
  });
});

// Governorates list
app.get('/api/governorates', (req, res) => {
  res.json(governorates);
});

app.get('/api/provinces', (req, res) => {
  res.json(governorates); // Same as governorates
});

// Stats
app.get('/api/stats', (req, res) => {
  const stats = {
    totalCandidates: candidates.length,
    byProvince: {},
    byParty: {},
    byGender: {
      Male: candidates.filter(c => c.gender === 'male').length,
      Female: candidates.filter(c => c.gender === 'female').length
    },
    byEducation: {}
  };
  
  res.json(stats);
});

const PORT = process.env.PORT || 10000;
app.listen(PORT, () => {
  console.log(`✅ Server running on port ${PORT}`);
});
```

**Deploy to Railway:**
```bash
npm init -y
npm install express cors
railway login
railway init
railway up
```

---

### Step 5: Verify Frontend Works (15 minutes)

**Test all candidate-related pages:**

1. **Home page** (`/en`)
   - [ ] Page loads without errors
   - [ ] AI post generation works
   - [ ] Feed displays (even if mock data)

2. **Candidates page** (`/en/candidates`)
   - [ ] Candidate cards display
   - [ ] Pagination works
   - [ ] Filters work (governorate, gender)
   - [ ] Search works

3. **Governorates page** (`/en/governorates`)
   - [ ] Governorate list displays
   - [ ] Clicking a governorate filters candidates

4. **Stats page** (`/en/stats`)
   - [ ] Statistics display correctly
   - [ ] Charts render properly

5. **Individual candidate** (`/en/candidates/1`)
   - [ ] Candidate details load
   - [ ] Profile information displays

**Test in all languages:**
- [ ] English (`/en/candidates`)
- [ ] Arabic (`/ar/candidates`) - verify RTL layout
- [ ] Kurdish (`/ku/candidates`)

---

### Step 6: Production Build Test (10 minutes)

Verify the app builds for production without errors:

```bash
npm run build
```

**Expected Output:**
```
✓ Compiled successfully
✓ Linting and checking validity of types
✓ Collecting page data
✓ Generating static pages (15/15)
✓ Finalizing page optimization

Route (app)                              Size     First Load JS
┌ ○ /[lang]                              142 kB        182 kB
├ ○ /[lang]/candidates                   8.2 kB        145 kB
└ ○ /[lang]/governorates                 1.4 kB        138 kB
```

**If build fails:**
- Check error messages
- Usually missing exports or type errors
- Fix and retry

---

## ✅ SUCCESS CRITERIA

Phase 1 is complete when:

- [x] No build errors
- [x] All candidate pages load
- [x] Data displays (real or mock)
- [x] No console errors
- [x] Tests pass in all 3 languages
- [x] Production build succeeds

---

## 🚨 TROUBLESHOOTING

### Error: "Cannot find module 'fetchCandidates'"

**Cause:** Export not added to `lib/api.ts`

**Fix:** Follow Step 1 above exactly

---

### Error: "API request failed with status 500"

**Cause:** Backend is broken or misconfigured

**Fix:** Use Option A (mock data) from Step 4

---

### Error: "CORS policy blocked"

**Cause:** Backend not allowing your frontend domain

**Fix:** Update backend CORS settings:
```javascript
app.use(cors({
  origin: [
    'http://localhost:3000',
    'https://your-domain.vercel.app'
  ]
}));
```

---

### Error: "getServerSideProps is not supported"

**Cause:** Using wrong Next.js pattern (Pages Router vs App Router)

**Fix:** This project uses App Router - use `async` Server Components instead

---

## 📊 PROGRESS TRACKING

**Day 1:**
- [ ] Step 1: Add API exports (30 min)
- [ ] Step 2: Test dev build (5 min)
- [ ] Step 3: Test Railway API (15 min)
- [ ] Step 4: Handle backend issues (60 min)

**Day 2:**
- [ ] Step 5: Verify all pages (15 min)
- [ ] Step 6: Production build test (10 min)
- [ ] Fix any remaining issues (variable)

**Day 3 (if needed):**
- [ ] Polish and final testing
- [ ] Documentation updates
- [ ] Deploy to Vercel/Cloudflare

---

## 🎉 NEXT STEPS

After Phase 1 is complete:
1. Document what works vs what doesn't
2. Prepare for Phase 2 (Backend Development)
3. Set up staging environment
4. Begin building social features backend

---

## 📞 NEED HELP?

**Common issues and solutions:**
- Check the main TECHNICAL_ANALYSIS_REPORT.md
- Review existing documentation
- Test each step in isolation
- Use console.log() liberally
- Check browser DevTools Network tab

---

**Phase 1 Complete = Working Candidate Browsing System** ✅

This establishes your foundation for building the social features in Phase 2-3.

**Let's go! 🚀**
