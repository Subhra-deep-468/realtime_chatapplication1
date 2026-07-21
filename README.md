# Chatly 💬

A real-time one-to-one chat application built with the MERN stack and Socket.io. Users can sign up, search for other users, chat instantly with text and images, and see who's online in real time — similar in concept to WhatsApp Web.

---

## Features

- **Authentication** — secure signup/login with JWT stored in HTTP-only cookies
- **Real-time messaging** — instant message delivery via Socket.io, no polling or refresh
- **Online status** — live online/offline indicators for all users
- **Image sharing** — send images in chat and set a profile picture, stored via Cloudinary
- **User search** — search for other registered users by name or username
- **Responsive UI** — mobile-friendly layout that toggles between sidebar and chat view
- **Emoji support** — built-in emoji picker for messages

---

## Tech Stack

### Frontend
| Technology | Purpose |
|---|---|
| React (Vite) | Component-based UI, fast dev experience |
| Redux Toolkit | Global state management (user, messages, socket, online status) |
| React Router DOM | Client-side routing |
| Axios | HTTP requests with cookie-based auth (`withCredentials`) |
| socket.io-client | Real-time bidirectional communication |
| Tailwind CSS | Utility-first styling |
| emoji-picker-react | In-chat emoji picker |
| react-icons | Icon set |

### Backend
| Technology | Purpose |
|---|---|
| Node.js + Express | REST API server |
| MongoDB + Mongoose | Database and ODM |
| Socket.io | Real-time messaging and online-status broadcasting |
| JWT (jsonwebtoken) | Stateless authentication |
| bcryptjs | Password hashing |
| Multer | Multipart form-data / file upload handling |
| Cloudinary | Persistent image hosting (profile pictures, chat images) |
| cookie-parser | Reading/writing HTTP-only auth cookies |
| cors | Cross-origin request handling between frontend and backend domains |
| dotenv | Environment variable management |

---

## Architecture Overview

```
React Frontend  ⇄  REST API (Express)  ⇄  MongoDB (Mongoose)
      ⇅                    ⇅
socket.io-client ⇄  Socket.io server (same HTTP server as Express)
                            ⇅
                      Cloudinary (image storage)
```

A single Node process runs both the REST API and the Socket.io server — Socket.io attaches directly to the same underlying `http.Server` that Express uses, so `server.listen()` (not `app.listen()`) starts both together.

### Database Schema

- **User** — `name`, `userName` (unique), `email` (unique), `password` (hashed), `image`
- **Message** — `sender`, `receiver`, `message`, `image`
- **Conversation** — `participants` (2 users), `messages` (array of Message references)

A dedicated `Conversation` document groups all messages between two users, so an entire chat thread can be fetched in a single `populate()` query instead of scanning the full Message collection.

---

## How It Works

1. **Sign up / log in** — password is hashed with bcrypt; a JWT (7-day expiry) is generated and returned as an HTTP-only, secure cookie.
2. **Protected routes** pass through an `isAuth` middleware that verifies the JWT from the cookie on every request.
3. **On login**, the frontend opens a Socket.io connection (passing the user's ID). The server maintains an in-memory `userId → socketId` map and broadcasts the live online-users list to everyone.
4. **Selecting a contact** fetches the full conversation history from MongoDB via `Conversation.findOne(...).populate("messages")`.
5. **Sending a message** — text and/or image are sent as `FormData`. Images are uploaded to Cloudinary first (via Multer for temporary handling), then the message is saved to MongoDB and linked to (or creates) a Conversation.
6. **Real-time delivery** — right after saving, the server looks up the receiver's active socket and emits a `newMessage` event directly to them, so the message appears instantly with no refresh. If the receiver is offline, the message stays safely in MongoDB for next time.

---

## Project Structure

```
chatly/
├── backend/
│   ├── config/          # DB connection, JWT token gen, Cloudinary config
│   ├── controllers/      # auth, user, message logic
│   ├── middlewares/       # isAuth (JWT verification), multer (file upload)
│   ├── models/            # User, Message, Conversation schemas
│   ├── routes/            # auth, user, message API routes
│   ├── socket/            # Socket.io server + online-user tracking
│   └── index.js            # entry point
│
└── frontend/
    ├── src/
    │   ├── components/     # SideBar, MessageArea, Sender/ReceiverMessage
    │   ├── customHooks/    # getCurrentUser, getOtherUsers, getMessages
    │   ├── pages/           # Login, SignUp, Home, Profile
    │   ├── redux/            # userSlice, messageSlice, store
    │   ├── App.jsx
    │   └── main.jsx
    └── ...
```

---

## Getting Started

### Prerequisites
- Node.js
- MongoDB (local or Atlas)
- Cloudinary account

### Backend Setup
```bash
cd backend
npm install
```

Create a `.env` file in `backend/`:
```
PORT=5000
MONGODB_URL=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret
CLOUD_NAME=your_cloudinary_cloud_name
API_KEY=your_cloudinary_api_key
API_SECRET=your_cloudinary_api_secret
```

Run the server:
```bash
npm run dev
```

### Frontend Setup
```bash
cd frontend
npm install
npm run dev
```

Update `serverUrl` in `frontend/src/main.jsx` to point to your backend URL if different from the default.

---

## API Endpoints

### Auth (`/api/auth`)
| Method | Route | Description |
|---|---|---|
| POST | `/signup` | Register a new user |
| POST | `/login` | Log in an existing user |
| GET | `/logout` | Clear auth cookie |

### User (`/api/user`)
| Method | Route | Description |
|---|---|---|
| GET | `/current` | Get logged-in user's details |
| GET | `/others` | Get all other users |
| PUT | `/profile` | Update name / profile picture |
| GET | `/search?query=` | Search users by name/username |

### Message (`/api/message`)
| Method | Route | Description |
|---|---|---|
| POST | `/send/:receiver` | Send a message (text/image) to a user |
| GET | `/get/:receiver` | Get full conversation with a user |

---

## Known Limitations / Future Improvements

- Online-user tracking is stored in memory on a single server instance — would need Redis (or a similar shared store) to scale across multiple instances.
- No pagination on message history; the entire conversation loads at once.
- No rate-limiting on login/signup endpoints.
- No message read-receipts or typing indicators yet.

---

## License

This project is open for educational and personal use.
