# Server-Side Rendering & Next.js: L4 Engineering Guide

## Part 1: SSR Fundamentals & Benefits

### 1.1 SSR vs CSR vs Static Generation

**Client-Side Rendering (CSR):** Server sends empty HTML, client downloads JavaScript and renders. Slower initial load, better interactivity after.

**Server-Side Rendering (SSR):** Server renders HTML on request, sends complete page to client. Faster initial load, SEO-friendly, more server resources.

**Static Generation:** Server pre-renders HTML at build time, serves static files. Fastest, best for blogs/marketing sites, can't handle dynamic per-user content.

```jsx
// CSR: empty HTML, client renders
// initial HTML: <div id="root"></div>
// JavaScript downloads and renders React tree

// SSR example: server renders
import ReactDOM from "react-dom/server";

const htmlString = ReactDOM.renderToString(<App user={user} />);
// HTML sent to client with data injected

// Static generation: build-time rendering
// HTML pre-generated, served from CDN
// Good for: blogs, documentation, marketing sites
```

### 1.2 Hydration & State Serialization

**Hydration** attaches React event listeners to pre-rendered HTML. Server sends HTML + JavaScript; client hydrates (adds interactivity) without re-rendering.

**Challenge:** ensuring server and client render identically. Mismatch causes hydration errors.

```jsx
// Server: render to string
function HomePage({ user }) {
  return <div>Welcome, {user.name}</div>;
}

const html = ReactDOM.renderToString(<HomePage user={user} />);
// Sends: <div>Welcome, Alice</div>

// Client: hydrate with same props
ReactDOM.hydrateRoot(
  document.getElementById("root"),
  <HomePage user={user} />
);

// Passing data from server to client
// Server: inject data in <script> tag
const dataScript = `window.__INITIAL_STATE__ = ${JSON.stringify({ user, posts: [] })}`;
const html = `
  <!DOCTYPE html>
  <script>${dataScript}</script>
  <div id="root">${htmlString}</div>
  <script src="/bundle.js"></script>
`;

// Client: read initial state
function App() {
  const [state] = React.useState(window.__INITIAL_STATE__ || {});
  return <HomePage {...state} />;
}

// Hydration mismatch: common error
// ❌ Problem: different on server vs client
function Component() {
  const [isMounted, setIsMounted] = React.useState(false);
  return isMounted ? "Client" : "Server"; // Hydration mismatch!
}

// ✅ Solution
function Component() {
  const [isMounted, setIsMounted] = React.useState(false);
  React.useEffect(() => {
    setIsMounted(true);
  }, []);
  if (!isMounted) return null; // Skip rendering until hydrated
  return "Client";
}
```

---

## Part 2: Next.js & Advanced SSR Patterns

### 2.1 Next.js Pages & Data Fetching

**Next.js** provides frameworks for SSR, static generation, and API routes. **getStaticProps** for static generation, **getServerSideProps** for SSR, **getStaticPaths** for dynamic routes.

```jsx
// Static generation: build-time rendering
export async function getStaticProps() {
  const posts = await fetch("https://api.example.com/posts");
  return {
    props: { posts },
    revalidate: 60 // ISR: revalidate every 60s
  };
}

export default function Blog({ posts }) {
  return <div>{posts.map(p => <article key={p.id}>{p.title}</article>)}</div>;
}

// Server-side rendering: request-time rendering
export async function getServerSideProps({ params, query, req, res }) {
  const user = await fetch(`/api/users/${params.id}`);
  return {
    props: { user }
  };
}

export default function UserPage({ user }) {
  return <div>{user.name}</div>;
}

// Dynamic routes with static generation
export async function getStaticPaths() {
  const posts = await fetch("/api/posts");
  const paths = posts.map(p => ({
    params: { slug: p.slug }
  }));
  return {
    paths,
    fallback: "blocking" // On-demand generation for unlisted paths
  };
}

export async function getStaticProps({ params }) {
  const post = await fetch(`/api/posts/${params.slug}`);
  return {
    props: { post },
    revalidate: 3600 // Revalidate hourly
  };
}

export default function Post({ post }) {
  return <article>{post.content}</article>;
}
```

### 2.2 Incremental Static Regeneration (ISR)

**ISR** revalidates static pages on-demand, enabling "static with revalidation." On request after revalidation period, Next.js re-generates page in background, serving stale content meanwhile.

```jsx
// ISR: static that updates periodically
export async function getStaticProps() {
  const product = await db.product.findOne({ id: 123 });
  return {
    props: { product },
    revalidate: 60 // Revalidate every 60 seconds
  };
}

export default function ProductPage({ product }) {
  return <div>{product.name}: ${product.price}</div>;
}

// On-demand revalidation
export default function Admin() {
  const handleRevalidate = async (productId) => {
    const res = await fetch(`/api/revalidate?secret=${process.env.REVALIDATE_SECRET}&path=/product/${productId}`, {
      method: "POST"
    });
  };
  return <button onClick={() => handleRevalidate(123)}>Revalidate Product</button>;
}

// Revalidation API route
export default async function handler(req, res) {
  // Verify secret
  if (req.query.secret !== process.env.REVALIDATE_SECRET) {
    return res.status(401).json({ message: "Invalid secret" });
  }

  try {
    // Revalidate paths
    await res.revalidate(req.query.path);
    return res.json({ revalidated: true });
  } catch (err) {
    return res.status(500).json({ message: "Error revalidating" });
  }
}
```

### 2.3 Performance & SEO Optimization

**Performance:** Next.js optimizes images, enables automatic code splitting, prefetches routes.

**SEO:** Build-time rendered HTML is crawlable; include metadata via Next.js Head or next/head.

```jsx
// Image optimization
import Image from "next/image";

export default function Hero() {
  return (
    <Image
      src="/banner.jpg"
      alt="Hero banner"
      width={1200}
      height={600}
      priority // Load immediately (above fold)
    />
  );
}

// SEO metadata
import Head from "next/head";

export default function Post({ post }) {
  return (
    <>
      <Head>
        <title>{post.title}</title>
        <meta name="description" content={post.excerpt} />
        <meta property="og:title" content={post.title} />
        <meta property="og:description" content={post.excerpt} />
        <meta property="og:image" content={post.image} />
      </Head>
      <article>{post.content}</article>
    </>
  );
}

// Canonical URLs (prevent duplicate content)
<Head>
  <link rel="canonical" href={`https://example.com/post/${post.slug}`} />
</Head>

// Structured data (JSON-LD)
<Head>
  <script type="application/ld+json">
    {JSON.stringify({
      "@context": "https://schema.org",
      "@type": "BlogPosting",
      headline: post.title,
      datePublished: post.publishedAt,
      author: { "@type": "Person", name: post.author }
    })}
  </script>
</Head>

// Route prefetching
import Link from "next/link";

export default function Nav() {
  return (
    <nav>
      <Link href="/about" prefetch={true}>About</Link> {/* Prefetch chunk */}
    </nav>
  );
}
```

---

## Interview Questions

**Q1: Explain CSR, SSR, and Static Generation. When would you use each?**

**CSR:** JavaScript renders client-side; slower initial load, better interactivity. Use for dashboards, user-specific content.

**SSR:** Server renders on request; faster initial load, SEO-friendly, more CPU. Use for content sites, SEO-critical pages.

**Static:** HTML pre-generated at build; fastest, best for CDN delivery. Use for blogs, marketing sites. Trade-off: can't handle dynamic per-user content.

**Q2: What is hydration and what's a common hydration error?**

Hydration attaches event listeners to server-rendered HTML. Server and client must render identically or hydration errors occur. Common cause: using client-only values during SSR (e.g., `window` object, random numbers). Solution: detect mount client-side, render null server-side.

**Q3: Explain Incremental Static Regeneration (ISR) and its benefits.**

ISR revalidates static pages on-demand. After revalidation period expires, Next.js re-generates page in background while serving stale content. Benefits: combines static performance with dynamic content updates, no expensive server rendering on every request.

**Q4: How do you optimize SEO in Next.js?**

Use static generation or SSR (build-time/request-time HTML is crawlable). Inject metadata via Head (title, description, OG tags). Include structured data (JSON-LD). Use canonical URLs to prevent duplicate content. Optimize images and performance (Core Web Vitals).

---

## Key Takeaways

1. **CSR slower initial load, SSR faster load, Static fastest** - Choose based on content type and update frequency
2. **Hydration attaches React to server-rendered HTML** - Server and client must render identically to avoid errors
3. **Next.js provides getStaticProps for build-time rendering** - Fast, cached, best for static content
4. **getServerSideProps for request-time rendering** - Fresh data per request, slower, more flexible
5. **ISR enables static with revalidation** - Best of both: fast static + periodic content updates
6. **Image optimization and code splitting improve performance** - Next.js handles automatically
7. **Metadata and structured data improve SEO** - Server-rendered HTML is crawlable
8. **Canonical URLs prevent duplicate content** - Important for SEO when content accessible via multiple URLs