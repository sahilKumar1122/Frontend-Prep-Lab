# System Design Questions

## Table of Contents
- [Design a News Feed](#design-a-news-feed)
- [Design an Autocomplete](#design-an-autocomplete)
- [Design a Image Gallery](#design-a-image-gallery)
- [Design a Chat Application](#design-a-chat-application)

---

## Design a News Feed

### Design a social media news feed (like Facebook/Twitter)

**Requirements:**
- Display posts from users
- Infinite scrolling
- Like/comment functionality
- Real-time updates
- Performance optimization

**Solution Approach:**

```javascript
// 1. Data Structure
interface Post {
  id: string;
  userId: string;
  content: string;
  timestamp: Date;
  likes: number;
  comments: Comment[];
  media?: Media[];
}

// 2. API Design
GET /api/feed?page=1&limit=20
POST /api/posts
PUT /api/posts/:id/like
POST /api/posts/:id/comments

// 3. Component Architecture
- NewsFeed (container)
  - PostList
    - Post (individual post)
      - PostHeader
      - PostContent
      - PostActions
      - CommentList
  - InfiniteScroll
  - CreatePost

// 4. State Management
const feed = {
  posts: [],
  loading: false,
  hasMore: true,
  page: 1
};

// 5. Implementation
function NewsFeed() {
  const [posts, setPosts] = useState([]);
  const [page, setPage] = useState(1);
  const [loading, setLoading] = useState(false);
  
  // Infinite scroll
  useEffect(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting && !loading) {
          loadMorePosts();
        }
      },
      { threshold: 1.0 }
    );
    
    return () => observer.disconnect();
  }, []);
  
  // Real-time updates (WebSocket)
  useEffect(() => {
    const ws = new WebSocket('ws://api.com/feed');
    
    ws.onmessage = (event) => {
      const newPost = JSON.parse(event.data);
      setPosts(prev => [newPost, ...prev]);
    };
    
    return () => ws.close();
  }, []);
  
  // Optimistic updates
  const handleLike = async (postId) => {
    // Update UI immediately
    setPosts(prev => prev.map(post =>
      post.id === postId
        ? { ...post, likes: post.likes + 1 }
        : post
    ));
    
    try {
      await api.likePost(postId);
    } catch (error) {
      // Revert on error
      setPosts(prev => prev.map(post =>
        post.id === postId
          ? { ...post, likes: post.likes - 1 }
          : post
      ));
    }
  };
}

// 6. Performance Optimizations
- Virtualization for long lists
- Image lazy loading
- Debounced scroll handlers
- Memoization (React.memo, useMemo)
- Code splitting
- CDN for media
- Pagination/Infinite scroll
- Caching strategies
```

**Key Considerations:**
- Scalability
- Real-time updates
- Performance
- User experience
- Error handling
- Offline support

---

## Design an Autocomplete

### Design a search autocomplete (like Google search)

**Requirements:**
- Show suggestions as user types
- Fast and responsive
- Handle typos
- Prioritize popular searches
- Keyboard navigation

**Solution:**

```javascript
// 1. Component Structure
function Autocomplete() {
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState([]);
  const [selectedIndex, setSelectedIndex] = useState(-1);
  const [loading, setLoading] = useState(false);
  
  // Debounced search
  const debouncedSearch = useMemo(
    () => debounce(async (searchQuery) => {
      if (!searchQuery) {
        setSuggestions([]);
        return;
      }
      
      setLoading(true);
      try {
        const results = await fetchSuggestions(searchQuery);
        setSuggestions(results);
      } catch (error) {
        console.error(error);
      } finally {
        setLoading(false);
      }
    }, 300),
    []
  );
  
  // Handle input change
  const handleChange = (e) => {
    const value = e.target.value;
    setQuery(value);
    debouncedSearch(value);
  };
  
  // Keyboard navigation
  const handleKeyDown = (e) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setSelectedIndex(prev =>
          prev < suggestions.length - 1 ? prev + 1 : prev
        );
        break;
      case 'ArrowUp':
        e.preventDefault();
        setSelectedIndex(prev => prev > 0 ? prev - 1 : -1);
        break;
      case 'Enter':
        if (selectedIndex >= 0) {
          selectSuggestion(suggestions[selectedIndex]);
        }
        break;
      case 'Escape':
        setSuggestions([]);
        setSelectedIndex(-1);
        break;
    }
  };
  
  return (
    <div className="autocomplete">
      <input
        type="text"
        value={query}
        onChange={handleChange}
        onKeyDown={handleKeyDown}
        placeholder="Search..."
        aria-label="Search"
        aria-autocomplete="list"
        aria-controls="suggestions"
      />
      {loading && <Spinner />}
      {suggestions.length > 0 && (
        <ul id="suggestions" role="listbox">
          {suggestions.map((suggestion, index) => (
            <li
              key={suggestion.id}
              role="option"
              aria-selected={index === selectedIndex}
              className={index === selectedIndex ? 'selected' : ''}
              onClick={() => selectSuggestion(suggestion)}
            >
              {highlightMatch(suggestion.text, query)}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}

// Utility functions
function debounce(func, wait) {
  let timeout;
  return function executedFunction(...args) {
    clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
}

function highlightMatch(text, query) {
  const regex = new RegExp(`(${query})`, 'gi');
  return text.replace(regex, '<mark>$1</mark>');
}

// API with caching
const cache = new Map();

async function fetchSuggestions(query) {
  // Check cache first
  if (cache.has(query)) {
    return cache.get(query);
  }
  
  const response = await fetch(`/api/autocomplete?q=${query}`);
  const data = await response.json();
  
  // Cache results
  cache.set(query, data);
  
  return data;
}
```

**Optimizations:**
- Debouncing (300ms)
- Caching results
- Cancel previous requests
- Lazy loading
- Trie data structure for fast lookup
- Edge cases handling

---

## Design a Image Gallery

### Design a photo gallery with lazy loading and lightbox

**Solution:**

```javascript
// 1. Component Structure
function ImageGallery({ images }) {
  const [selectedImage, setSelectedImage] = useState(null);
  const [visibleImages, setVisibleImages] = useState([]);
  const observerRef = useRef();
  
  // Lazy loading with Intersection Observer
  useEffect(() => {
    observerRef.current = new IntersectionObserver(
      (entries) => {
        entries.forEach(entry => {
          if (entry.isIntersecting) {
            const img = entry.target;
            img.src = img.dataset.src;
            img.classList.add('loaded');
            observerRef.current.unobserve(img);
          }
        });
      },
      { rootMargin: '50px' }
    );
    
    return () => observerRef.current?.disconnect();
  }, []);
  
  return (
    <>
      <div className="gallery">
        {images.map((image, index) => (
          <ImageCard
            key={image.id}
            image={image}
            onClick={() => setSelectedImage(index)}
            observerRef={observerRef}
          />
        ))}
      </div>
      
      {selectedImage !== null && (
        <Lightbox
          images={images}
          currentIndex={selectedImage}
          onClose={() => setSelectedImage(null)}
        />
      )}
    </>
  );
}

// Image card with lazy loading
function ImageCard({ image, onClick, observerRef }) {
  const imgRef = useRef();
  
  useEffect(() => {
    if (imgRef.current && observerRef.current) {
      observerRef.current.observe(imgRef.current);
    }
  }, []);
  
  return (
    <div className="image-card" onClick={onClick}>
      <img
        ref={imgRef}
        data-src={image.url}
        src={image.thumbnail}
        alt={image.alt}
        loading="lazy"
      />
    </div>
  );
}

// Lightbox component
function Lightbox({ images, currentIndex, onClose }) {
  const [index, setIndex] = useState(currentIndex);
  
  useEffect(() => {
    const handleKeyDown = (e) => {
      switch (e.key) {
        case 'Escape':
          onClose();
          break;
        case 'ArrowLeft':
          setIndex(prev => Math.max(0, prev - 1));
          break;
        case 'ArrowRight':
          setIndex(prev => Math.min(images.length - 1, prev + 1));
          break;
      }
    };
    
    document.addEventListener('keydown', handleKeyDown);
    return () => document.removeEventListener('keydown', handleKeyDown);
  }, []);
  
  return (
    <div className="lightbox" onClick={onClose}>
      <button onClick={onClose}>×</button>
      <button onClick={() => setIndex(i => Math.max(0, i - 1))}>
        ‹
      </button>
      <img
        src={images[index].url}
        alt={images[index].alt}
        onClick={(e) => e.stopPropagation()}
      />
      <button onClick={() => setIndex(i => Math.min(images.length - 1, i + 1))}>
        ›
      </button>
    </div>
  );
}
```

---

## Design a Chat Application

### Design a real-time chat application

**Requirements:**
- Real-time messaging
- Online status
- Typing indicators
- Message history
- Unread counts

**Solution:**

```javascript
// WebSocket connection
class ChatService {
  constructor() {
    this.ws = null;
    this.listeners = new Map();
  }
  
  connect(userId) {
    this.ws = new WebSocket(`ws://api.com/chat?user=${userId}`);
    
    this.ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      this.emit(data.type, data.payload);
    };
    
    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
      this.reconnect();
    };
  }
  
  send(type, payload) {
    this.ws.send(JSON.stringify({ type, payload }));
  }
  
  on(event, callback) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event).push(callback);
  }
  
  emit(event, data) {
    const callbacks = this.listeners.get(event) || [];
    callbacks.forEach(cb => cb(data));
  }
}

// Chat component
function ChatApp() {
  const [messages, setMessages] = useState([]);
  const [onlineUsers, setOnlineUsers] = useState([]);
  const [typing, setTyping] = useState({});
  const chatService = useRef(new ChatService());
  
  useEffect(() => {
    const service = chatService.current;
    service.connect(currentUserId);
    
    service.on('message', (message) => {
      setMessages(prev => [...prev, message]);
    });
    
    service.on('typing', ({ userId, isTyping }) => {
      setTyping(prev => ({ ...prev, [userId]: isTyping }));
    });
    
    service.on('online', (users) => {
      setOnlineUsers(users);
    });
    
    return () => service.disconnect();
  }, []);
  
  const sendMessage = (text) => {
    chatService.current.send('message', {
      text,
      timestamp: Date.now()
    });
  };
  
  const handleTyping = debounce(() => {
    chatService.current.send('typing', { isTyping: true });
    setTimeout(() => {
      chatService.current.send('typing', { isTyping: false });
    }, 3000);
  }, 500);
  
  return (
    <div className="chat-app">
      <Sidebar users={onlineUsers} />
      <MessageList messages={messages} typing={typing} />
      <MessageInput onSend={sendMessage} onTyping={handleTyping} />
    </div>
  );
}
```

**Key Features:**
- WebSocket for real-time communication
- Optimistic UI updates
- Message persistence
- Reconnection logic
- Typing indicators
- Online status
- Unread counters
