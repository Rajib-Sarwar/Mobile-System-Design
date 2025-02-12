-------------------------------------------------
🎯 1️⃣ Fetching Initial Posts (On App Load)
-------------------------------------------------
FeedViewController  
 ├──► FeedPresenter.getInitialPosts()  
      ├──► FeedRepository.getPosts()  
           ├──► (If Online) Fetch from FeedAPIService  
           ├──► (If Offline) Fetch from LocalStorageManager  
           └──► Return posts to FeedPresenter  
      ├──► FeedPresenter updates FeedViewController  
      └──► UI updates with fetched posts  

-------------------------------------------------
⚡ 2️⃣ Getting New Posts in Real-Time (Using SSE)
-------------------------------------------------
FeedViewController  
 ├──► FeedPresenter.startListeningForNewPosts()  
      ├──► FeedRepository.startListeningForNewPosts()  
           ├──► SSEManager.connect() (Opens SSE connection)  
           ├──► Server pushes new post via SSE  
           ├──► SSEManager notifies FeedRepository  
           ├──► FeedRepository saves post in LocalStorageManager  
           ├──► FeedRepository informs FeedPresenter  
           └──► FeedPresenter updates FeedViewController  
      ├──► UI updates with new post  
      └──► New post appears in the feed  

-------------------------------------------------
👍 3️⃣ Handling Like Action (Online Mode)
-------------------------------------------------
FeedViewController  
 ├──► FeedPresenter.handleLike(postId, liked)  
      ├──► FeedRepository.saveLike(postId, liked)  
      ├──► FeedRepository → WebSocketManager.sendLike(postId, liked)  
      ├──► Server updates like count & broadcasts update  
      ├──► WebSocketManager receives updated like count  
      ├──► WebSocketManager → FeedRepository.updateLike()  
      ├──► FeedRepository → FeedPresenter.updateLikeUI()  
      ├──► FeedPresenter updates FeedViewController  
      └──► Like count updates in real-time  

-------------------------------------------------
📴 4️⃣ Handling Offline Mode (Fetching & Storing Likes)
-------------------------------------------------
FeedViewController  
 ├──► FeedPresenter.fetchPosts()  
      ├──► FeedRepository.getPosts()  
           ├──► LocalStorageManager.getCachedPosts()  
           ├──► FeedRepository → FeedPresenter  
           ├──► FeedPresenter updates FeedViewController  
           └──► UI updates with cached posts  

FeedViewController  
 ├──► FeedPresenter.handleLike(postId, liked)  
      ├──► FeedPresenter → FeedRepository.storeOfflineLike(postId, liked)  
      ├──► Like action is queued in LocalStorageManager  
      ├──► UI updates immediately (optimistic update)  
      └──► Like is **not sent** to server until online  

-------------------------------------------------
🔄 5️⃣ Syncing Pending Likes (When Online)
-------------------------------------------------
FeedPresenter  
 ├──► FeedRepository.syncOfflineLikes()  
      ├──► LocalStorageManager.getPendingLikes()  
      ├──► FeedRepository → WebSocketManager.sendLike()  
      ├──► Server processes likes & updates counts  
      ├──► WebSocketManager receives like updates  
      ├──► WebSocketManager → FeedRepository.updateLike()  
      ├──► FeedRepository → FeedPresenter.updateLikeUI()  
      ├──► FeedPresenter updates FeedViewController  
      └──► LocalStorageManager clears synced likes  


To design the flow using SOLID principles, we need to ensure that the system is modular, easy to maintain, and scalable. I'll define the classes and protocols with the necessary functions for each part of the system. The focus will be on the design and relationships between the components.

### 1. `FeedViewController`

**Responsibilities:**
- Display the UI to the user.
- Delegate interaction events to the presenter.

```swift
protocol FeedViewControllerDelegate: AnyObject {
    func didLikePost(postId: String, liked: Bool)
}

class FeedViewController: UIViewController {
    var presenter: FeedPresenterProtocol?
    
    func displayInitialPosts(_ posts: [Post]) {
        // Display posts on UI
    }
    
    func displayNewPost(_ post: Post) {
        // Display new post on UI
    }
    
    func updateLikeUI(postId: String, liked: Bool) {
        // Update the like count UI
    }
    
    func startListeningForNewPosts() {
        presenter?.startListeningForNewPosts()
    }
    
    func fetchPosts() {
        presenter?.fetchPosts()
    }
    
    func handleLike(postId: String, liked: Bool) {
        presenter?.handleLike(postId: postId, liked: liked)
    }
}
```

---

### 2. `FeedPresenterProtocol`

**Responsibilities:**
- Coordinate between the `FeedViewController`, `FeedRepository`, and other components.
- Ensure correct UI updates and data flow.

```swift
protocol FeedPresenterProtocol: AnyObject {
    func fetchPosts()
    func startListeningForNewPosts()
    func handleLike(postId: String, liked: Bool)
    func updateLikeUI(postId: String, liked: Bool)
}
```

---

### 3. `FeedPresenter`

**Responsibilities:**
- Manage the flow of data between the `FeedViewController` and the repository.
- Handle business logic for fetching posts, likes, and real-time updates.

```swift
class FeedPresenter: FeedPresenterProtocol {
    var viewController: FeedViewController?
    var repository: FeedRepositoryProtocol?
    
    func fetchPosts() {
        repository?.getPosts()
    }
    
    func startListeningForNewPosts() {
        repository?.startListeningForNewPosts()
    }
    
    func handleLike(postId: String, liked: Bool) {
        repository?.saveLike(postId: postId, liked: liked)
    }
    
    func updateLikeUI(postId: String, liked: Bool) {
        viewController?.updateLikeUI(postId: postId, liked: liked)
    }
}
```

---

### 4. `FeedRepositoryProtocol`

**Responsibilities:**
- Provide methods to interact with data sources (e.g., API, local storage).
- Implement methods for fetching posts, handling likes, and managing real-time updates.

```swift
protocol FeedRepositoryProtocol: AnyObject {
    func getPosts()
    func startListeningForNewPosts()
    func saveLike(postId: String, liked: Bool)
    func storeOfflineLike(postId: String, liked: Bool)
    func syncOfflineLikes()
}
```

---

### 5. `FeedRepository`

**Responsibilities:**
- Manage interaction with the data sources.
- Fetch posts from the network or local storage.
- Handle the saving of likes, offline syncing, and real-time updates.

```swift
class FeedRepository: FeedRepositoryProtocol {
    var feedAPIService: FeedAPIServiceProtocol?
    var localStorageManager: LocalStorageManagerProtocol?
    var sseManager: SSEManagerProtocol?
    var webSocketManager: WebSocketManagerProtocol?
    
    func getPosts() {
        // Fetch posts from FeedAPIService or LocalStorageManager based on network status
    }
    
    func startListeningForNewPosts() {
        sseManager?.connect()
    }
    
    func saveLike(postId: String, liked: Bool) {
        webSocketManager?.sendLike(postId: postId, liked: liked)
    }
    
    func storeOfflineLike(postId: String, liked: Bool) {
        localStorageManager?.storeOfflineLike(postId: postId, liked: liked)
    }
    
    func syncOfflineLikes() {
        localStorageManager?.getPendingLikes()
        // Send likes via WebSocketManager once back online
    }
}
```

---

### 6. `FeedAPIServiceProtocol`

**Responsibilities:**
- Fetch data from the feed API.
  
```swift
protocol FeedAPIServiceProtocol: AnyObject {
    func fetchPosts(completion: @escaping ([Post]) -> Void)
}
```

---

### 7. `SSEManagerProtocol`

**Responsibilities:**
- Manage Server-Sent Events (SSE) connection and push new posts.

```swift
protocol SSEManagerProtocol: AnyObject {
    func connect()
    func listenForNewPosts(completion: @escaping (Post) -> Void)
}
```

---

### 8. `WebSocketManagerProtocol`

**Responsibilities:**
- Handle WebSocket communication for real-time updates (likes).

```swift
protocol WebSocketManagerProtocol: AnyObject {
    func sendLike(postId: String, liked: Bool)
    func receiveLikeUpdates(completion: @escaping (String, Bool) -> Void)
}
```

---

### 9. `LocalStorageManagerProtocol`

**Responsibilities:**
- Manage local storage for caching posts and offline data.
- Save and retrieve posts and likes from local storage.

```swift
protocol LocalStorageManagerProtocol: AnyObject {
    func getCachedPosts() -> [Post]
    func storeOfflineLike(postId: String, liked: Bool)
    func getPendingLikes() -> [(postId: String, liked: Bool)]
    func clearSyncedLikes()
}
```

---

### 10. `Post`

**Responsibilities:**
- A data model representing a post.

```swift
struct Post {
    let postId: String
    var content: String
    var likeCount: Int
}
```

---

### Key Notes:

1. **Single Responsibility Principle (SRP)**: Each class or protocol is focused on a single task. For example, `FeedViewController` is responsible only for displaying the UI, and `FeedRepository` manages data fetching and saving.
   
2. **Open/Closed Principle (OCP)**: New features, like handling different data sources or real-time updates, can be added by extending existing protocols or implementing new ones without modifying existing code.

3. **Liskov Substitution Principle (LSP)**: All concrete implementations should conform to their respective protocols, ensuring that they can be substituted for one another without affecting the system.

4. **Interface Segregation Principle (ISP)**: Smaller, focused protocols ensure that classes only implement the functionality they need. For example, the `WebSocketManagerProtocol` only has methods related to WebSocket interactions.

5. **Dependency Inversion Principle (DIP)**: High-level modules (`FeedPresenter`, `FeedViewController`) depend on abstractions (`FeedRepositoryProtocol`, `SSEManagerProtocol`), not concrete implementations. This ensures flexibility and easy substitution of components.

This design maintains a clear separation of concerns while adhering to SOLID principles for scalability and maintainability.
