Project Structure
server/
│
├── src
│
├── controllers
│      blog.controller.ts
│      ai.controller.ts
│
├── services
│      blog/
│           create.blog.service.ts
│
│      ai/
│           chat.blog.service.ts
│
├── repositories
│      blog.repository.ts
│
├── lib
│      gemini
│          embedding.ts
│          chunk.ts
│          retrieve.ts
│
├── routes
│      blog.routes.ts
│      ai.routes.ts
STEP 1 — User Creates a Blog

Next.js

POST /api/blog

Body

{
    "title":"Understanding React Hooks",
    "content":"React Hooks allow..."
}

↓

Express Route

router.post("/blog", blogController.create)
STEP 2 — Controller
export async function create(req,res){

    const response =
        await createBlogService(req.body);

    res.json(response);

}

Nothing special.

STEP 3 — Service
export async function createBlogService(data){

    const blog =
        await blogRepository.create(data);

    await indexBlog(blog);

    return blog;

}

Notice

Saving

and

Indexing

are separate.

Save Blog

↓

Index Blog
STEP 4 — Repository
await prisma.blog.create({

data:{

title,

content

}

});

Returns

{
    "id":"blog-1",
    "title":"Understanding React Hooks",
    "content":"React Hooks..."
}
STEP 5 — Index Blog

This is where RAG begins.

indexBlog(blog)
export async function indexBlog(blog){

    const chunks =
        chunkText(blog.content);

    ...
}

Input

React Hooks...

useState...

useEffect...

useMemo...
STEP 6 — chunkText()
export function chunkText(

text:string,

size=1000,

overlap=200

){

const chunks=[];

let start=0;

while(start<text.length){

const end=Math.min(

start+size,

text.length

);

chunks.push(

text.slice(start,end)

);

start+=size-overlap;

}

return chunks;

}

Output

[
"React Hooks allow...",

"useState is...",

"useEffect is...",

"useMemo..."
]
STEP 7 — Generate Embedding

Loop

for(const chunk of chunks){

const embedding=

await generateEmbedding(chunk);

}

Function

export async function generateEmbedding(text:string){

const result=

await genAI.models.embedContent({

model:"gemini-embedding-001",

contents:text

});

return result.embeddings[0].values;

}

Input

useEffect is used...

Output

[
0.122,

0.884,

...

768 values
]
STEP 8 — Save Chunk

Loop

for(

let i=0;

i<chunks.length;

i++

){

const embedding=

await generateEmbedding(

chunks[i]

);

await prisma.blogChunk.create({

data:{

blogId:blog.id,

chunkIndex:i,

content:chunks[i],

embedding

}

});

}

Database

BlogChunk

chunk	embedding
React Hooks	768 values
useState	768 values
useEffect	768 values
useMemo	768 values

Done.

Knowledge Base ready.

User asks chatbot

Next.js

POST /chat

Body

{

"question":

"What blogs mention useEffect?"

}
Controller
chatController.ask()

↓

Service

chatBlogService(question)
STEP 9 — Embed Question
const questionEmbedding=

await generateEmbedding(

question

);

Input

How does useEffect work?

Output

[
0.341,

0.664,

...

768
]
STEP 10 — Retrieve Similar Chunks
const chunks=

await retrieveChunks(

questionEmbedding

);

Function

export async function retrieveChunks(

embedding:number[]

){

return prisma.$queryRaw`

SELECT *

FROM "BlogChunk"

ORDER BY embedding <=> ${embedding}

LIMIT 5

`;

}

Output

[
{

"content":"useEffect is used for side effects"

},

{

"content":"React Hooks allow..."

}
]
STEP 11 — Build Context
const context=

chunks

.map(

chunk=>chunk.content

)

.join("\n\n");

Output

useEffect is used for side effects.

React Hooks allow functional components...
STEP 12 — Gemini
const prompt=`

Answer ONLY using the context.

Context

${context}

Question

${question}

`;

Call Gemini

const result=

await genAI.models.generateContent({

model:

"gemini-2.5-flash",

contents:prompt

});

Gemini returns

useEffect is used to execute side effects after rendering such as fetching data, timers, and subscriptions.

Return to frontend.
