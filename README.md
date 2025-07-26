# AstersTech-Week3-Demo2
---

## Step 1: Get Your GROQ API Key

1. Go to [https://groq.com](https://groq.com) and **sign up** or **log in**.  
2. Navigate to your account dashboard and find the section to generate or view your **API Key**.  
3. Copy your **API Key** — you will need this for your backend integration.

---

## Step 2: Install the GROQ SDK

In your project root, run:

//Make sure to replace "your-project-file-name" with your actual root file name

```bash
cd "your-project-file-name"
npm install groq-sdk axios
```
---

## Step 3: Set Environment Variables
Create a ".env.local" file at your project root with:

Replace "your_groq_api_key_here" with the key you copied from Groq.

```js
GROQ_API_KEY=your_groq_api_key_here
```
---
## Step 4: Create a GROQ API Route
First create a new folder called "groq" in your api's folder (the one in your app folder).
Then create a new file called "route.js" in your groq folder (app/api/groq/route.js):

```js
import { Groq } from 'groq-sdk';

// Get your API key from environment variables (make sure it's set in your .env file)
const GROQ_API_KEY = process.env.GROQ_API_KEY;

export async function POST(req) {
  try {
    // Get the JSON data sent from the frontend (we expect a "message" property)
    const { message } = await req.json();

    // If no message was sent, return an error response
    if (!message) {
      return new Response(
        JSON.stringify({ error: 'No message provided' }), 
        { status: 400 } // 400 means bad request
      );
    }

    // Create a new Groq client instance using your API key
    const groq = new Groq({ apiKey: GROQ_API_KEY });

    // Ask the Groq API to generate a reply based on the message
    const chatCompletion = await groq.chat.completions.create({
      messages: [
        // The system message tells the assistant how to behave
        { role: 'system', content: 'You are a helpful assistant.' },
        // The user's message is what we want a response for
        { role: 'user', content: message },
      ],
      model: 'meta-llama/llama-4-scout-17b-16e-instruct', // The AI model to use
      temperature: 1,              // Controls randomness: 1 = normal, 0 = deterministic
      max_completion_tokens: 1024, // Max length of the reply
      top_p: 1,                   // Controls diversity of the output
      stream: false,              // false means wait for full response, no partial streaming
      stop: null,                 // No special stopping condition
    });

    // Extract the text reply from the API response
    const reply = chatCompletion.choices[0]?.message?.content || 'No response.';

    // Send back the reply as JSON with status 200 (OK)
    return new Response(
      JSON.stringify({ reply }),
      {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
      }
    );
  } catch (error) {
    // If something goes wrong, log the error for debugging
    console.error('Groq API Error:', error);

    // Return a generic error message with status 500 (server error)
    return new Response(
      JSON.stringify({ error: 'Something went wrong' }),
      { status: 500 }
    );
  }
}
```
---
## Step 5: Add ChatBot to Your Protected Page
Now we will be connecting the Groq route you created in your backend to your front end. We will do this on the protected page you created last week. Open you protected page file and follow the code below.

```jsx
import { useEffect, useState } from 'react';
import axios from 'axios'; // NEW: For sending POST requests to the backend GROQ API
import { onAuthStateChanged, signOut } from 'firebase/auth';
import { auth } from '../firebase';
import { useRouter } from 'next/router';

export default function Protected() {
  const router = useRouter();
  const [user, setUser] = useState(null);

  const [query, setQuery] = useState('');
  
  // NEW: Store chat messages between user and GROQ bot
  const [messages, setMessages] = useState([]);

  // NEW: Loading state to disable input during API call
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      if (!currentUser) router.push('/login');
      else setUser(currentUser);
    });
    return () => unsubscribe();
  }, [router]);

  const handleLogout = async () => {
    await signOut(auth);
    router.push('/login');
  };

  // NEW: Send user query to backend GROQ API and update chat messages
  const sendQuery = async (e) => {
    e.preventDefault();
    if (!query.trim()) return;

    // Add user's message to chat history immediately
    const userMessage = { sender: 'user', text: query };
    setMessages((prev) => [...prev, userMessage]);

    setLoading(true);

    try {
      // POST request to backend API with user's query
      const res = await axios.post('/api/groq', { message: query });

      // Add bot's reply to chat history (response from GROQ)
      const botMessage = {
        sender: 'bot',
        text: res.data.reply,
      };
      setMessages((prev) => [...prev, botMessage]);
    } catch (error) {
      // Show error message if API call fails
      setMessages((prev) => [
        ...prev,
        { sender: 'bot', text: '❌ Error fetching data from GROQ.' },
      ]);
    } finally {
      setLoading(false);
      setQuery(''); // Clear input after sending
    }
  };

  return (
    <div className="min-h-screen bg-green-50 p-6 flex flex-col items-center">
      <div className="max-w-xl w-full bg-white rounded shadow p-6 mb-4">
        <h1 className="text-xl font-bold mb-2 text-center">🔒 Protected Page</h1>
        {user && <p className="mb-4 text-center">Welcome, <strong>{user.email}</strong>!</p>}
        <button
          onClick={handleLogout}
          className="mb-6 bg-red-500 text-white py-2 px-4 rounded hover:bg-red-600"
        >
          Logout
        </button>

        {/* NEW: Chat message area */}
        <div className="mb-4 max-h-64 overflow-y-auto border p-4 rounded bg-gray-100">
          {/* Show placeholder if no messages yet */}
          {messages.length === 0 && <p className="text-center text-gray-500">Ask a GROQ query below...</p>}

          {/* Display each message with different styles for user and bot */}
          {messages.map((msg, i) => (
            <pre
              key={i}
              className={`whitespace-pre-wrap mb-2 p-2 rounded ${
                msg.sender === 'user' ? 'bg-blue-200 text-blue-900' : 'bg-gray-200 text-gray-900'
              }`}
            >
              <strong>{msg.sender === 'user' ? 'You' : 'GROQ Bot'}:</strong> {msg.text}
            </pre>
          ))}
        </div>

        {/* NEW: Input form for GROQ queries */}
        <form onSubmit={sendQuery} className="flex space-x-2">
          <input
            type="text"
            placeholder="Enter GROQ query"
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            className="flex-grow p-2 border rounded"
            disabled={loading} // Disable input while waiting for bot reply
          />
          <button
            type="submit"
            disabled={loading} // Disable button while loading
            className="bg-blue-500 text-white py-2 px-4 rounded hover:bg-blue-600 disabled:opacity-50"
          >
            Send
          </button>
        </form>
      </div>
    </div>
  );
}
```
