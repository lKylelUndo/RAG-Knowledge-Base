📚 Blog RAG (Retrieval-Augmented Generation) Flow
📁 Project Structure
server/
└── src/
    ├── controllers/
    │   ├── blog.controller.ts
    │   └── ai.controller.ts
    │
    ├── services/
    │   ├── blog/
    │   │   ├── create.blog.service.ts
    │   │   └── index.blog.service.ts
    │   │
    │   └── ai/
    │       └── chat.blog.service.ts
    │
    ├── repositories/
    │   └── blog.repository.ts
    │
    ├── lib/
    │   └── gemini/
    │       ├── chunk.ts
    │       ├── embedding.ts
    │       ├── retrieve.ts
    │       └── prompt.ts
    │
    └── routes/
        ├── blog.routes.ts
        └── ai.routes.ts
Phase 1 — Building the Knowledge Base

This process happens only once whenever an administrator creates or updates a blog.

Admin
   │
   ▼
Create Blog
   │
   ▼
Save Blog
   │
   ▼
Chunk Blog Content
   │
   ▼
Generate Embeddings
   │
   ▼
Store Chunks + Embeddings
Step 1 — Create Blog
Next.js Request
POST /api/blog
Request Body
{
  "title": "Understanding React Hooks",
  "content": "React Hooks allow functional components..."
}
Step 2 — Express Route
router.post("/blog", blogController.create);
Step 3 — Controller
export async function create(req, res) {
    const response = await createBlogService(req.body);
    return res.json(response);
}
Step 4 — Service
export async function createBlogService(data) {

    const blog = await blogRepository.create(data);

    await indexBlog(blog);

    return blog;

}

Notice that the service has two responsibilities:

Save Blog
      │
      ▼
Index Blog
Step 5 — Repository
await prisma.blog.create({
    data: {
        title,
        content
    }
});
Database Result
id	title	content
blog-1	Understanding React Hooks	Full blog content
Phase 2 — Indexing the Blog

After saving the blog, the content is converted into searchable knowledge.

Step 6 — Split Blog into Chunks
const chunks = chunkText(blog.content);
Function
export function chunkText(
    text: string,
    size = 1000,
    overlap = 200
) {
    const chunks = [];

    let start = 0;

    while (start < text.length) {

        const end = Math.min(start + size, text.length);

        chunks.push(text.slice(start, end));

        start += size - overlap;
    }

    return chunks;
}
Output
[
  "React Hooks allow functional components...",
  "useState manages component state...",
  "useEffect handles side effects...",
  "useMemo optimizes expensive calculations..."
]
Step 7 — Generate Embeddings

For every chunk:

for (const chunk of chunks) {
    const embedding = await generateEmbedding(chunk);
}
Function
export async function generateEmbedding(text: string) {

    const result = await genAI.models.embedContent({
        model: "gemini-embedding-001",
        contents: text
    });

    return result.embeddings[0].values;
}
Output
[
  0.124,
  -0.552,
  0.772,
  ...
]

(768-dimensional vector)

Step 8 — Save Chunks
for (let i = 0; i < chunks.length; i++) {

    const embedding = await generateEmbedding(chunks[i]);

    await prisma.blogChunk.create({
        data: {
            blogId: blog.id,
            chunkIndex: i,
            content: chunks[i],
            embedding
        }
    });

}
BlogChunk Table
Chunk Index	Content	Embedding
0	React Hooks...	Vector
1	useState...	Vector
2	useEffect...	Vector
3	useMemo...	Vector

The knowledge base is now ready.

Phase 3 — User Asks the Chatbot
Visitor
   │
   ▼
Ask Question
   │
   ▼
Generate Question Embedding
   │
   ▼
Vector Similarity Search
   │
   ▼
Retrieve Top Chunks
   │
   ▼
Build Context
   │
   ▼
Gemini Generates Answer
Step 9 — User Request
POST /chat
Request
{
  "question": "How does useEffect work?"
}
Step 10 — Generate Question Embedding
const questionEmbedding = await generateEmbedding(question);

Output

[
  0.412,
  0.123,
  ...
]
Step 11 — Retrieve Similar Chunks
const chunks = await retrieveChunks(questionEmbedding);
Function
export async function retrieveChunks(
    embedding: number[]
) {

    return prisma.$queryRaw`
        SELECT *
        FROM "BlogChunk"
        ORDER BY embedding <=> ${embedding}
        LIMIT 5
    `;

}
Retrieved Chunks
[
  {
    "content": "useEffect is used for side effects."
  },
  {
    "content": "React Hooks allow functional components."
  }
]
Step 12 — Build Context
const context = chunks
    .map(chunk => chunk.content)
    .join("\n\n");

Generated Context

useEffect is used for side effects.

React Hooks allow functional components.
Step 13 — Ask Gemini
const prompt = `
Answer ONLY using the provided context.

Context:
${context}

Question:
${question}
`;

const result = await genAI.models.generateContent({
    model: "gemini-2.5-flash",
    contents: prompt
});
Gemini Response
useEffect is a React Hook used for executing side effects after rendering, such as fetching data, subscriptions, and timers.
Complete RAG Pipeline
                 INDEXING PHASE (One-Time)

Admin
   │
   ▼
Create Blog
   │
   ▼
Save Blog
   │
   ▼
Chunk Blog
   │
   ▼
Generate Embeddings
   │
   ▼
Store Chunks + Embeddings


                 RETRIEVAL PHASE (Every Chat)

Visitor Question
      │
      ▼
Generate Question Embedding
      │
      ▼
Vector Similarity Search
      │
      ▼
Retrieve Top 5 Chunks
      │
      ▼
Build Context
      │
      ▼
Gemini Generates Grounded Answer
      │
      ▼
Return Response

This format is much more suitable for a README because it clearly separates the Indexing Phase (building the knowledge base) from the Retrieval Phase (answering user questions), making the RAG workflow easy for reviewers or your thesis panel to understand.
