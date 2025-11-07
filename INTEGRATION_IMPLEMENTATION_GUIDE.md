# 🔧 INTEGRATION IMPLEMENTATION GUIDE
## Iraqi Digital Democracy Platform - Critical Fixes & Backend Integration

**Target:** Production deployment in 5 days  
**Priority:** CRITICAL PATH FIXES  
**Difficulty:** Medium  

---

## 🎯 OVERVIEW

This guide provides **exact code** to fix the critical integration issues identified in the comprehensive analysis. Follow these steps sequentially to make the application production-ready.

---

## ⚠️ CRITICAL ISSUE #1: Missing Data Fetching Functions

**Problem:** The application imports server-side fetch functions that don't exist:
```typescript
// These imports FAIL:
import { fetchCandidates, fetchStats, fetchGovernorates } from '@/lib/api';
```

**Impact:** Application won't build or run  
**Time to Fix:** 4-6 hours  
**Priority:** CRITICAL (blocking deployment)

### 📝 Solution: Add Missing Functions to `lib/api.ts`

**File:** `/lib/api.ts`

Add the following code at the END of the file (after the candidatesAPI export):

```typescript
// ============================================================================
// SERVER-SIDE DATA FETCHING FUNCTIONS (Next.js App Router)
// ============================================================================

/**
 * Fetch candidates with filters and pagination (Server-side)
 * Use this in Server Components (app directory)
 */
export async function fetchCandidates(filters: {
  page?: number;
  limit?: number;
  query?: string;
  governorate?: string;
  gender?: 'male' | 'female';
} = {}): Promise<PaginatedCandidates> {
  try {
    const params = new URLSearchParams();
    
    if (filters.page) params.set('page', filters.page.toString());
    if (filters.limit) params.set('limit', filters.limit.toString());
    if (filters.query) params.set('query', filters.query);
    if (filters.governorate) params.set('governorate', filters.governorate);
    if (filters.gender) params.set('gender', filters.gender);

    const url = `${API_BASE_URL}/api/candidates?${params.toString()}`;
    
    const response = await fetch(url, {
      next: { revalidate: 300 }, // Revalidate every 5 minutes
      headers: {
        'Content-Type': 'application/json',
      },
    });

    if (!response.ok) {
      console.error(`Failed to fetch candidates: ${response.status} ${response.statusText}`);
      // Return empty result instead of throwing
      return {
        data: [],
        total: 0,
        page: filters.page || 1,
        limit: filters.limit || 12,
      };
    }

    const data = await response.json();
    
    // Map backend fields to frontend format (if needed)
    const mappedData = {
      data: data.data.map((candidate: any) => ({
        id: candidate.id.toString(),
        name: candidate.name,
        name_ar: candidate.nameArabic || candidate.name_ar,
        name_ku: candidate.nameKurdish || candidate.name_ku,
        photo: candidate.photo,
        bio: candidate.biography || candidate.bio,
        bio_ar: candidate.biographyArabic || candidate.bio_ar,
        bio_ku: candidate.biographyKurdish || candidate.bio_ku,
        party: candidate.party,
        governorate: candidate.province || candidate.governorate,
        age: candidate.age,
        gender: candidate.gender?.toLowerCase() as 'male' | 'female',
        verified: candidate.verified || false,
        created_at: candidate.created_at || new Date().toISOString(),
        updated_at: candidate.updated_at || new Date().toISOString(),
      })),
      total: data.total,
      page: data.page || filters.page || 1,
      limit: data.limit || filters.limit || 12,
    };

    return mappedData;
  } catch (error) {
    console.error('Error in fetchCandidates:', error);
    // Return empty result instead of throwing to prevent page crashes
    return {
      data: [],
      total: 0,
      page: filters.page || 1,
      limit: filters.limit || 12,
    };
  }
}

/**
 * Fetch election statistics (Server-side)
 */
export async function fetchStats(): Promise<Stats> {
  try {
    const response = await fetch(`${API_BASE_URL}/api/stats`, {
      next: { revalidate: 3600 }, // Revalidate every hour
      headers: {
        'Content-Type': 'application/json',
      },
    });

    if (!response.ok) {
      throw new Error(`Failed to fetch stats: ${response.statusText}`);
    }

    const data = await response.json();
    
    // Return data in expected format
    return {
      total_candidates: data.totalCandidates || data.total_candidates || 0,
      total_parties: data.totalParties || data.total_parties || 0,
      total_governorates: data.totalGovernorates || data.total_governorates || 18,
      last_updated: data.lastUpdated || data.last_updated || new Date().toISOString(),
      gender_distribution: {
        Male: data.byGender?.male || data.gender_distribution?.Male || 0,
        Female: data.byGender?.female || data.gender_distribution?.Female || 0,
      },
      candidates_per_governorate: (data.byProvince || data.candidates_per_governorate || []).map((item: any) => ({
        governorate_name: item.province || item.governorate_name || item.name,
        candidate_count: item.count || item.candidate_count || 0,
      })),
    };
  } catch (error) {
    console.error('Error in fetchStats:', error);
    // Return default stats to prevent page crash
    return {
      total_candidates: 0,
      total_parties: 0,
      total_governorates: 18,
      last_updated: new Date().toISOString(),
      gender_distribution: {
        Male: 0,
        Female: 0,
      },
      candidates_per_governorate: [],
    };
  }
}

/**
 * Fetch list of governorates (Server-side)
 */
export async function fetchGovernorates(): Promise<Governorate[]> {
  try {
    const response = await fetch(`${API_BASE_URL}/api/provinces`, {
      next: { revalidate: 86400 }, // Revalidate once per day (static data)
      headers: {
        'Content-Type': 'application/json',
      },
    });

    if (!response.ok) {
      console.error(`Failed to fetch governorates: ${response.statusText}`);
      return getDefaultGovernorates();
    }

    const data = await response.json();
    
    // Map to frontend format
    return data.map((gov: any) => ({
      id: gov.id?.toString() || gov.name,
      name: gov.name || gov.province,
      name_ar: gov.nameArabic || gov.name_ar || gov.name,
      name_ku: gov.nameKurdish || gov.name_ku || gov.name,
      population: gov.population,
      region: gov.region || 'central',
    }));
  } catch (error) {
    console.error('Error in fetchGovernorates:', error);
    return getDefaultGovernorates();
  }
}

/**
 * Fallback governorates list (in case backend is unavailable)
 */
function getDefaultGovernorates(): Governorate[] {
  return [
    { id: '1', name: 'Baghdad', name_ar: 'بغداد', name_ku: 'بەغدا', region: 'central' },
    { id: '2', name: 'Basra', name_ar: 'البصرة', name_ku: 'بەسرە', region: 'south' },
    { id: '3', name: 'Nineveh', name_ar: 'نينوى', name_ku: 'نەینەوا', region: 'north' },
    { id: '4', name: 'Erbil', name_ar: 'أربيل', name_ku: 'هەولێر', region: 'kurdistan' },
    { id: '5', name: 'Sulaymaniyah', name_ar: 'السليمانية', name_ku: 'سلێمانی', region: 'kurdistan' },
    { id: '6', name: 'Dohuk', name_ar: 'دهوك', name_ku: 'دهۆک', region: 'kurdistan' },
    { id: '7', name: 'Najaf', name_ar: 'النجف', name_ku: 'نەجەف', region: 'south' },
    { id: '8', name: 'Karbala', name_ar: 'كربلاء', name_ku: 'کەربەلا', region: 'south' },
    { id: '9', name: 'Anbar', name_ar: 'الأنبار', name_ku: 'ئەنبار', region: 'west' },
    { id: '10', name: 'Diyala', name_ar: 'ديالى', name_ku: 'دیالە', region: 'central' },
    { id: '11', name: 'Kirkuk', name_ar: 'كركوك', name_ku: 'کەرکووک', region: 'north' },
    { id: '12', name: 'Saladin', name_ar: 'صلاح الدين', name_ku: 'سەلاحەدین', region: 'north' },
    { id: '13', name: 'Wasit', name_ar: 'واسط', name_ku: 'واسیت', region: 'central' },
    { id: '14', name: 'Maysan', name_ar: 'ميسان', name_ku: 'مەیسان', region: 'south' },
    { id: '15', name: 'Dhi Qar', name_ar: 'ذي قار', name_ku: 'زی قار', region: 'south' },
    { id: '16', name: 'Muthanna', name_ar: 'المثنى', name_ku: 'موسەنا', region: 'south' },
    { id: '17', name: 'Qadisiyyah', name_ar: 'القادسية', name_ku: 'قادسیە', region: 'south' },
    { id: '18', name: 'Babil', name_ar: 'بابل', name_ku: 'بابل', region: 'central' },
  ];
}

/**
 * Fetch single candidate by ID (Server-side)
 * Use this for the candidate detail page
 */
export async function fetchCandidateById(id: string): Promise<Candidate | null> {
  try {
    const response = await fetch(`${API_BASE_URL}/api/candidates/${id}`, {
      next: { revalidate: 300 }, // Revalidate every 5 minutes
      headers: {
        'Content-Type': 'application/json',
      },
    });

    if (!response.ok) {
      console.error(`Failed to fetch candidate ${id}: ${response.statusText}`);
      return null;
    }

    const candidate = await response.json();
    
    // Map to frontend format
    return {
      id: candidate.id.toString(),
      name: candidate.name,
      name_ar: candidate.nameArabic || candidate.name_ar,
      name_ku: candidate.nameKurdish || candidate.name_ku,
      photo: candidate.photo,
      bio: candidate.biography || candidate.bio,
      bio_ar: candidate.biographyArabic || candidate.bio_ar,
      bio_ku: candidate.biographyKurdish || candidate.bio_ku,
      party: candidate.party,
      governorate: candidate.province || candidate.governorate,
      age: candidate.age,
      gender: candidate.gender?.toLowerCase() as 'male' | 'female',
      education: candidate.education,
      experience: candidate.experience,
      platform: candidate.platform || candidate.manifesto,
      verified: candidate.verified || false,
      created_at: candidate.created_at || new Date().toISOString(),
      updated_at: candidate.updated_at || new Date().toISOString(),
    };
  } catch (error) {
    console.error(`Error fetching candidate ${id}:`, error);
    return null;
  }
}

/**
 * Like/unlike a post (Client-side only)
 * Call this from Client Components with 'use client'
 */
export async function likePost(postId: string): Promise<{ success: boolean }> {
  try {
    // TODO: Implement when backend endpoint is ready
    // For now, return success (mock)
    console.log(`Liking post ${postId} (mock implementation)`);
    return { success: true };
  } catch (error) {
    console.error(`Error liking post ${postId}:`, error);
    return { success: false };
  }
}
```

**Add these TypeScript interfaces to `lib/types.ts` if missing:**

```typescript
export interface PaginatedCandidates {
  data: Candidate[];
  total: number;
  page: number;
  limit: number;
}
```

---

## ⚠️ CRITICAL ISSUE #2: Missing Candidate Detail Page

**Problem:** No page exists to view individual candidate profiles  
**Impact:** Users can see list but can't view full candidate details  
**Time to Fix:** 6-8 hours  
**Priority:** HIGH

### 📝 Solution: Create Candidate Detail Page

**File:** `/app/[lang]/candidates/[id]/page.tsx` (NEW FILE)

```typescript
import { fetchCandidateById } from '@/lib/api';
import { Locale } from '@/lib/i18n-config';
import { getDictionary } from '@/lib/dictionaries';
import { Metadata } from 'next';
import Image from 'next/image';
import Link from 'next/link';
import { notFound } from 'next/navigation';
import { User, Landmark, GraduationCap, Briefcase, ArrowLeft, Share2, ExternalLink } from 'lucide-react';
import React from 'react';

type Props = {
  params: { lang: Locale; id: string };
};

export async function generateMetadata({ params: { lang, id } }: Props): Promise<Metadata> {
  const candidate = await fetchCandidateById(id);
  const dictionary = await getDictionary(lang);

  if (!candidate) {
    return {
      title: `${dictionary.notFound.title} | ${dictionary.metadata.title}`,
    };
  }

  const name = lang === 'ar' && candidate.name_ar ? candidate.name_ar : candidate.name;

  return {
    title: `${name} | ${dictionary.metadata.title}`,
    description: candidate.bio || dictionary.page.candidates.description,
  };
}

export default async function CandidateDetailPage({ params: { lang, id } }: Props) {
  const dictionary = await getDictionary(lang);
  const candidate = await fetchCandidateById(id);

  if (!candidate) {
    notFound();
  }

  const name = lang === 'ar' && candidate.name_ar ? candidate.name_ar : candidate.name;
  const bio = lang === 'ar' && candidate.bio_ar ? candidate.bio_ar : candidate.bio;
  const party = candidate.party;
  const governorate = candidate.governorate;

  return (
    <div className="min-h-screen bg-gray-50 dark:bg-gray-900 py-8">
      <div className="container mx-auto px-4 sm:px-6 lg:px-8 max-w-4xl">
        {/* Back Button */}
        <Link
          href={`/${lang}/candidates`}
          className="inline-flex items-center gap-2 text-green-600 hover:text-green-700 dark:text-green-400 mb-6 transition"
        >
          <ArrowLeft size={20} />
          <span>{dictionary.navigation?.candidates || 'Back to Candidates'}</span>
        </Link>

        {/* Candidate Header */}
        <div className="bg-white dark:bg-gray-800 rounded-2xl shadow-lg overflow-hidden">
          <div className="md:flex">
            {/* Profile Image */}
            <div className="md:w-1/3 flex items-center justify-center p-8 bg-gradient-to-br from-green-50 to-green-100 dark:from-gray-700 dark:to-gray-800">
              <Image
                src={candidate.photo || `https://avatar.iran.liara.run/public/${candidate.gender === 'female' ? 'girl' : 'boy'}?username=${candidate.id}`}
                alt={name}
                width={300}
                height={300}
                className="rounded-full object-cover shadow-xl"
              />
            </div>

            {/* Candidate Info */}
            <div className="md:w-2/3 p-8">
              <div className="flex items-start justify-between">
                <div>
                  <h1 className="text-4xl font-bold text-gray-900 dark:text-white mb-2">
                    {name}
                  </h1>
                  {candidate.verified && (
                    <span className="inline-flex items-center gap-1 text-green-600 dark:text-green-400 text-sm font-semibold">
                      <svg className="w-5 h-5" fill="currentColor" viewBox="0 0 20 20">
                        <path fillRule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clipRule="evenodd" />
                      </svg>
                      Verified Candidate
                    </span>
                  )}
                </div>
                <button
                  onClick={() => {
                    if (navigator.share) {
                      navigator.share({
                        title: name,
                        text: `${dictionary.page.profile?.shareText || 'Learn more about'} ${name}`,
                        url: window.location.href,
                      });
                    }
                  }}
                  className="p-2 rounded-lg hover:bg-gray-100 dark:hover:bg-gray-700 transition"
                  aria-label="Share"
                >
                  <Share2 size={20} />
                </button>
              </div>

              <div className="mt-6 space-y-4">
                {/* Party */}
                <div className="flex items-center gap-3 text-gray-700 dark:text-gray-300">
                  <div className="p-2 bg-green-100 dark:bg-green-900/30 rounded-lg">
                    <User size={20} className="text-green-600 dark:text-green-400" />
                  </div>
                  <div>
                    <div className="text-sm text-gray-500 dark:text-gray-400">Political Party</div>
                    <div className="font-semibold">{party}</div>
                  </div>
                </div>

                {/* Governorate */}
                <div className="flex items-center gap-3 text-gray-700 dark:text-gray-300">
                  <div className="p-2 bg-blue-100 dark:bg-blue-900/30 rounded-lg">
                    <Landmark size={20} className="text-blue-600 dark:text-blue-400" />
                  </div>
                  <div>
                    <div className="text-sm text-gray-500 dark:text-gray-400">Governorate</div>
                    <div className="font-semibold">{governorate}</div>
                  </div>
                </div>

                {/* Age & Gender */}
                {(candidate.age || candidate.gender) && (
                  <div className="flex items-center gap-3 text-gray-700 dark:text-gray-300">
                    <div className="p-2 bg-purple-100 dark:bg-purple-900/30 rounded-lg">
                      <User size={20} className="text-purple-600 dark:text-purple-400" />
                    </div>
                    <div>
                      <div className="text-sm text-gray-500 dark:text-gray-400">Demographics</div>
                      <div className="font-semibold">
                        {candidate.age && `${candidate.age} years old`}
                        {candidate.age && candidate.gender && ' • '}
                        {candidate.gender && (candidate.gender === 'male' ? dictionary.candidate?.male : dictionary.candidate?.female)}
                      </div>
                    </div>
                  </div>
                )}

                {/* Education */}
                {candidate.education && (
                  <div className="flex items-center gap-3 text-gray-700 dark:text-gray-300">
                    <div className="p-2 bg-yellow-100 dark:bg-yellow-900/30 rounded-lg">
                      <GraduationCap size={20} className="text-yellow-600 dark:text-yellow-400" />
                    </div>
                    <div>
                      <div className="text-sm text-gray-500 dark:text-gray-400">Education</div>
                      <div className="font-semibold">{candidate.education}</div>
                    </div>
                  </div>
                )}

                {/* Experience */}
                {candidate.experience && (
                  <div className="flex items-center gap-3 text-gray-700 dark:text-gray-300">
                    <div className="p-2 bg-red-100 dark:bg-red-900/30 rounded-lg">
                      <Briefcase size={20} className="text-red-600 dark:text-red-400" />
                    </div>
                    <div>
                      <div className="text-sm text-gray-500 dark:text-gray-400">Experience</div>
                      <div className="font-semibold">{candidate.experience}</div>
                    </div>
                  </div>
                )}
              </div>
            </div>
          </div>
        </div>

        {/* Biography */}
        {bio && (
          <div className="mt-8 bg-white dark:bg-gray-800 rounded-2xl shadow-lg p-8">
            <h2 className="text-2xl font-bold text-gray-900 dark:text-white mb-4">Biography</h2>
            <p className="text-gray-700 dark:text-gray-300 leading-relaxed whitespace-pre-wrap">
              {bio}
            </p>
          </div>
        )}

        {/* Platform/Manifesto */}
        {candidate.platform && (
          <div className="mt-8 bg-white dark:bg-gray-800 rounded-2xl shadow-lg p-8">
            <h2 className="text-2xl font-bold text-gray-900 dark:text-white mb-4">Campaign Platform</h2>
            <p className="text-gray-700 dark:text-gray-300 leading-relaxed whitespace-pre-wrap">
              {candidate.platform}
            </p>
          </div>
        )}

        {/* Contact / Social Media (if available) */}
        <div className="mt-8 bg-white dark:bg-gray-800 rounded-2xl shadow-lg p-8">
          <h2 className="text-2xl font-bold text-gray-900 dark:text-white mb-4">Get in Touch</h2>
          <p className="text-gray-600 dark:text-gray-400 mb-4">
            Connect with {name} to learn more about their campaign and vision for {governorate}.
          </p>
          <div className="flex gap-4">
            <button className="px-6 py-3 bg-green-600 text-white rounded-lg font-semibold hover:bg-green-700 transition">
              Contact Candidate
            </button>
            <Link
              href={`/${lang}/candidates`}
              className="px-6 py-3 border-2 border-gray-300 dark:border-gray-600 rounded-lg font-semibold hover:bg-gray-100 dark:hover:bg-gray-700 transition"
            >
              Compare Candidates
            </Link>
          </div>
        </div>
      </div>
    </div>
  );
}
```

---

## ⚠️ CRITICAL ISSUE #3: Social Feed Stub

**Problem:** Feed component is a placeholder  
**Impact:** Core social feature non-functional  
**Time to Fix:** 8-16 hours (backend + frontend)  
**Priority:** HIGH (after candidate features)

### 📝 Solution Part 1: Replace Feed Stub

**File:** `/components/social/Feed.tsx`

```typescript
'use client';

import React from 'react';
import { Post } from '@/lib/types';
import { Heart, MessageCircle, Share2, MoreHorizontal } from 'lucide-react';
import { formatDistanceToNow } from 'date-fns';
import Image from 'next/image';

type FeedProps = {
  lang: string;
  posts: Post[];
};

export default function Feed({ lang, posts }: FeedProps) {
  const [likedPosts, setLikedPosts] = React.useState<Set<string>>(new Set());

  const handleLike = (postId: string) => {
    setLikedPosts(prev => {
      const newSet = new Set(prev);
      if (newSet.has(postId)) {
        newSet.delete(postId);
      } else {
        newSet.add(postId);
      }
      return newSet;
    });
  };

  if (posts.length === 0) {
    return (
      <div className="text-center py-12">
        <p className="text-gray-500 dark:text-gray-400">
          No posts yet. Be the first to share something!
        </p>
      </div>
    );
  }

  return (
    <div className="space-y-6">
      {posts.map((post) => {
        const isLiked = likedPosts.has(post.id);
        const likeCount = post.likes + (isLiked ? 1 : 0);

        return (
          <article
            key={post.id}
            className="bg-white dark:bg-gray-800 rounded-2xl shadow-md overflow-hidden hover:shadow-lg transition-shadow"
          >
            {/* Post Header */}
            <div className="p-4 flex items-center justify-between">
              <div className="flex items-center gap-3">
                <Image
                  src={post.author.avatar}
                  alt={post.author.name}
                  width={48}
                  height={48}
                  className="rounded-full"
                />
                <div>
                  <div className="flex items-center gap-2">
                    <h3 className="font-semibold text-gray-900 dark:text-white">
                      {post.author.name}
                    </h3>
                    {post.author.verified && (
                      <svg className="w-5 h-5 text-green-600" fill="currentColor" viewBox="0 0 20 20">
                        <path fillRule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clipRule="evenodd" />
                      </svg>
                    )}
                  </div>
                  <p className="text-sm text-gray-500 dark:text-gray-400">
                    {formatDistanceToNow(new Date(post.timestamp), { addSuffix: true })}
                  </p>
                </div>
              </div>
              <button className="p-2 hover:bg-gray-100 dark:hover:bg-gray-700 rounded-full transition">
                <MoreHorizontal size={20} className="text-gray-500" />
              </button>
            </div>

            {/* Post Content */}
            <div className="px-4 pb-3">
              <p className="text-gray-900 dark:text-white whitespace-pre-wrap">
                {post.content}
              </p>
            </div>

            {/* Post Image */}
            {post.image && (
              <div className="relative w-full aspect-video">
                <Image
                  src={post.image}
                  alt="Post image"
                  fill
                  className="object-cover"
                />
              </div>
            )}

            {/* Post Actions */}
            <div className="px-4 py-3 border-t border-gray-200 dark:border-gray-700">
              <div className="flex items-center justify-between">
                <button
                  onClick={() => handleLike(post.id)}
                  className={`flex items-center gap-2 px-4 py-2 rounded-lg transition ${
                    isLiked
                      ? 'text-red-600 bg-red-50 dark:bg-red-900/20'
                      : 'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700'
                  }`}
                >
                  <Heart size={20} fill={isLiked ? 'currentColor' : 'none'} />
                  <span className="font-semibold">{likeCount}</span>
                </button>

                <button className="flex items-center gap-2 px-4 py-2 rounded-lg text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700 transition">
                  <MessageCircle size={20} />
                  <span className="font-semibold">{post.comments}</span>
                </button>

                <button className="flex items-center gap-2 px-4 py-2 rounded-lg text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700 transition">
                  <Share2 size={20} />
                  <span className="font-semibold">{post.shares}</span>
                </button>
              </div>
            </div>
          </article>
        );
      })}
    </div>
  );
}
```

---

## 🔄 TESTING CHECKLIST

After implementing the fixes above, test each feature:

### Day 1 Testing (After adding fetch functions)
- [ ] Run `npm run build` - Should succeed
- [ ] Visit `/en/candidates` - Should show candidates
- [ ] Try filters (governorate, gender) - Should work
- [ ] Check pagination - Should navigate pages
- [ ] Visit `/en/stats` - Should show statistics
- [ ] Check charts - Should render correctly
- [ ] Test Arabic layout `/ar/candidates` - RTL should work

### Day 2 Testing (After candidate detail page)
- [ ] Click on a candidate - Should open detail page
- [ ] Verify all candidate information displays
- [ ] Test back button - Should return to list
- [ ] Test share button - Should trigger share dialog
- [ ] Verify images load correctly
- [ ] Test on mobile - Should be responsive

### Day 3 Testing (After social feed)
- [ ] Create a post - Should appear in feed
- [ ] Like a post - Should increment count
- [ ] Unlike a post - Should decrement count
- [ ] Check post timestamps - Should show relative time
- [ ] Verify verified badges show correctly

---

## 📦 ENVIRONMENT SETUP

**Create `.env.local` file:**

```env
# Backend API (Railway)
NEXT_PUBLIC_API_BASE_URL=https://digitaldemocracy-iraq-production.up.railway.app

# AI Features (Google Gemini)
NEXT_PUBLIC_GEMINI_API_KEY=your-actual-gemini-api-key-here

# Optional: Backup API
NEXT_PUBLIC_BACKUP_API=https://winter-leaf-f532.safaribosafar.workers.dev
```

**Create `.env.example` for documentation:**

```env
# Backend API URL
NEXT_PUBLIC_API_BASE_URL=https://your-backend-url.up.railway.app

# Google Gemini API Key (for AI features)
# Get yours at: https://makersuite.google.com/app/apikey
NEXT_PUBLIC_GEMINI_API_KEY=your-gemini-api-key

# Optional: Fallback API
NEXT_PUBLIC_BACKUP_API=https://your-backup-api.workers.dev
```

---

## 🚀 DEPLOYMENT STEPS

### Vercel Deployment

1. **Push to GitHub:**
```bash
git add .
git commit -m "feat: add missing data fetching functions and candidate detail page"
git push origin future-features
```

2. **Deploy to Vercel:**
```bash
# Install Vercel CLI
npm i -g vercel

# Login
vercel login

# Deploy
vercel --prod

# Set environment variables
vercel env add NEXT_PUBLIC_API_BASE_URL
vercel env add NEXT_PUBLIC_GEMINI_API_KEY
```

3. **Verify Deployment:**
- Visit your Vercel URL
- Test candidate listing
- Test candidate detail pages
- Test statistics dashboard
- Verify all 3 languages work

---

## ⏱️ ESTIMATED TIMELINE

**Day 1 (6 hours):**
- [ ] Add fetch functions to lib/api.ts (4 hours)
- [ ] Test build and deployment (1 hour)
- [ ] Fix any errors (1 hour)

**Day 2 (8 hours):**
- [ ] Create candidate detail page (6 hours)
- [ ] Add error handling (1 hour)
- [ ] Test all features (1 hour)

**Day 3 (8 hours):**
- [ ] Implement social feed component (4 hours)
- [ ] Add like/share functionality (2 hours)
- [ ] Final testing (2 hours)

**Day 4 (4 hours):**
- [ ] Deploy to production
- [ ] Smoke testing
- [ ] Documentation updates

**Total: 26 hours (3.5 developer days)**

---

## 🆘 TROUBLESHOOTING

### Build Error: "Cannot find module fetchCandidates"
**Solution:** Make sure you added all fetch functions to lib/api.ts and exported them

### 404 Error on Candidate Detail
**Solution:** Verify the file path is exactly: `app/[lang]/candidates/[id]/page.tsx`

### Data Not Loading
**Solution:** Check Railway backend is running at: https://digitaldemocracy-iraq-production.up.railway.app/api/health

### Images Not Loading
**Solution:** Add image domains to `next.config.mjs`:
```javascript
images: {
  unoptimized: true,
  domains: ['avatar.iran.liara.run', 'i.pravatar.cc', 'picsum.photos']
}
```

---

## ✅ VERIFICATION

After implementing all fixes, verify:

```bash
# 1. Build succeeds
npm run build

# 2. No TypeScript errors
npx tsc --noEmit

# 3. Dev server runs
npm run dev

# 4. Test these URLs:
# http://localhost:3000/en/candidates
# http://localhost:3000/en/candidates/1
# http://localhost:3000/en/stats
# http://localhost:3000/ar/candidates (RTL test)
```

**Expected Result:** All pages load without errors, data displays correctly, RTL works properly.

---

## 📝 NEXT STEPS AFTER FIXES

1. **Add Authentication**
   - Implement JWT-based auth
   - Add login/register modals
   - Protect user-specific routes

2. **Implement Social Backend**
   - Create Prisma schema for posts
   - Implement POST /api/posts
   - Add like/comment endpoints

3. **Add Real-time Features**
   - WebSocket integration
   - Live notifications
   - Real-time feed updates

4. **Optimize Performance**
   - Add React Query for caching
   - Implement lazy loading
   - Optimize images

---

**END OF IMPLEMENTATION GUIDE**

For questions or issues, refer to the main COMPREHENSIVE_CODEBASE_ANALYSIS.md document.
