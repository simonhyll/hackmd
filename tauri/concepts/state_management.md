<style>
  p {
    text-align: justify;
  }
</style>

# Tauri Concepts: <br/>State Management
How to use a Tauri "State" and what it is exactly isn't covered very well in the current documentation, even though it's one of the most powerful tools you have in your Tauri arsenal.

## What is a "State"?
A State is any data structure you create in Rust with the goal of storing data for the duration of your applications runtime. Using a State you can get very easy access to the value both in commands as well as anywhere you have access to an AppHandle.

Before we get into the more nitty gritty details on how Tauri handles your State and how to use them, lets get a bit better understanding first for related topics that will be important for later on when we decide to use State's in practice.

## How does "serde" work with a State?
The name "serde" comes from the words "serialize" and "deserialize", and it's what the library helps you implement for your data structures.

Serialization is when you take a value from its original form and turn it into something else, usually a string. In Tauri's case this is what we use for turning your Rust value into a Javascript value. If you need to pass data from the backend to the frontend it has to be serializeable.

Deserialization is when you take a serialized value, usually a string, and turn it back into its original format, which in Rusts case is turning it back into a struct. In Tauri's case this is how we turn inputs to commands from Javascript back into their Rust format. If you need to pass data from the frontend to the backend it has to be deserializeable.

When you create a State you can choose to derive from Serialize, Deserialize or both, depending on which directions your struct needs to support.

Something I use a lot for my structs is the macro `#[serde(skip_serializing)]`. Lets say I have a struct called `AuthState`. In that state I'm keeping an authentication token. I however am building a very security critical application, so I don't want to pass the token to the frontend. However, keeping that token in a separate struct would be very annoying to do. So what I do is simply instruct serde that certain specific properties in the struct should be excluded from serialization, thus ensuring that I can return the `AuthState` safely to the frontend and serde will ensure that any security critical data is never passed back to the frontend.

```rust
use serde::Serialize;

#[derive(Serialize)]
pub(crate) struct AuthState {
    #[serde(skip_serializing)]
    token: Option<String>,
    logged_in: bool,
}
```
Here you can see me create an `AuthState` where the token is an `Option`, since we might not yet have a token to use, and in the event I want to communicate the current state of authentication with the frontend I have skipped serializing that property, meaning if we passed that struct over to the frontend then Javascript would receive an object like so `{ logged_in: false }`.

## What is a Mutex?
Mutex is short for "mutually exclusive". When you're passing a value between threads in Rust you will need to ensure that two threads aren't accessing the same value at the same time. To do this we wrap values in a Mutex so that we can lock that value to a single thread, and then unlock the value once the value is dropped.

If we try to manipulate the same value in multiple threads there will be issues.
```rust
let my_mutex = 42
std::thread::spawn(|| {
  my_lock = 69;
})
std::thread::spawn(|| {  
  my_lock = 24;
})
// The compiler saves you from this by complaining about moved values, but in
// theory what would happen here is that multiple threads might write to the same
// location in memory at the same time, causing the program to crash
```
If we instead use a Mutex we ensure that only one thread can mutate the state at the same time. Note that this code wouldn't work either because we need an Arc, but I'm going to talk about that right after this.
```rust
let my_mutex = Mutex::new(42);
std::thread::spawn(|| {
  let mut my_lock = my_mutex.lock().unwrap();
  my_lock = 69;
})
std::thread::spawn(|| {
  let mut my_lock = my_mutex.lock().unwrap();  
  my_lock = 24;
})
// The value can be 42, 69 or 24 now depending on various factors, but
// at least it can no longer crash
```

**Important note on locking**: Make sure you don't end up locking your thread forever.
```rust
// Locks forever because the lock is dropped at the end of the function
// only to immediately create another lock in the next iteration
loop {
  let my_val = my_mutex.lock().unwrap();
  std::thread::sleep(std::time::Duration::from_secs(1));
}
// Doesn't lock forever because the lock is dropped immediately
loop {
  std::thread::sleep(std::time::Duration::from_secs(1));  
  let my_val = my_mutex.lock().unwrap();
}
// Doesn't lock forever because we're manually dropping the lock
loop {
  let my_val = my_mutex.lock().unwrap();
  drop(my_val)
  std::thread::sleep(std::time::Duration::from_secs(1));  
}
```

## What is an Arc?
An Arc is a reference counter. Normally in Rust values only exist in one place at one time. What languages will normally do is keep a reference counter on a variable in order to know when all references to that value are gone so it knows that it's safe to drop. You can essentially see it as that in Rust that reference counter is by default just 1, you have a single reference to a value, once it's gone it's gone forever.

Using an Arc you can create a reference counter for your variable, allowing you to keep creating more references to the same value so that the original references value doesn't get cleaned up.

So if we look at the example from before with Mutex and add an Arc.
```rust
// Create an Arc<Mutex<i32>>
let my_mutex = Arc::new(Mutex::new(42));
// Create a new reference
let arced_mutex = Arc::clone(my_mutex);
std::thread::spawn(move || {
  // The reference is now moved to the thread, but the actual value remains
  // in the main thread
  let mut my_lock = arced_mutex.lock().unwrap();
  my_lock = 69;
})
// We construct one reference per thread
let arced_mutex = Arc::clone(my_mutex);
std::thread::spawn(|| {
  let mut my_lock = arced_mutex.lock().unwrap();
  my_lock = 24;
})
// When the threads finish executing their individual references are dropped
// This means the original Arc<Mutex>> no longer has any references to it
// Which means that once the original value is dropped it can finally
// be cleaned up
```
## Do we have to use Arc<Mutex<T>>?
The Arc allows us to get a reference to a value across threads, and a Mutex gives us the ability to access that value in a thread safe manner.

However, the State itself gives us something very close to what Arc gives us!

So why did we bother going over what an Arc is? Well, because if you create a State that needs to be accessed and manipulated anywhere that Tauri can't provide it to you easily, then you would create the Arc yourself.

So in most cases, if you don't have an advanced use-case, you may very well get away with just creating a `Mutex<MyState>>`

## Shortly about errors
I won't go too in-depth on error handling for commands here, that deserves its own article. This is the error I'm going to use for now. The most noteworthy things here are that for you to return an error from Rust to Javascript you'll need to implement serialization for that as well just like with your struct, and I'm implementing `PoisonError` for this error, which is an error that may be raised when you lock a `Mutex`.

```rust
#[derive(Debug, thiserror::Error)]
enum Error {
    #[error(transparent)]
    Io(#[from] std::io::Error),
    #[error("the mutex was poisoned")]
    PoisonError(String),
}

impl serde::Serialize for Error {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: serde::ser::Serializer,
    {
        serializer.serialize_str(self.to_string().as_ref())
    }
}

impl<T> From<std::sync::PoisonError<T>> for Error {
    fn from(err: std::sync::PoisonError<T>) -> Self {
        Error::PoisonError(err.to_string())
    }
}
```
## So what is a State?
TODO: Nitty gritty details on what a State is.

## Managing a State

```rust
use serde::Serialize;

#[derive(Serialize)]
pub(crate) struct AuthState {
    #[serde(skip_serializing)]
    token: Option<String>,
    logged_in: bool,
}

fn main() {
  tauri::Builder::default()
    .manage(Mutex::new())
    .run(tauri::generate_context!())
    .expect("failed to run app");
}
```

## Using the State

```rust
#[tauri::command]
async fn login(state_mutex: State<'_, Mutex<AuthState>>) -> Result<AuthState, Error> {
    println!("Logging in");
    let mut state = state_mutex.lock()?;
    state.logged_in = true;
    Ok(state.clone())
}
```

## When should I create a State?
- Is the information unique to the entire app and not just the window?
- Does the information have to be accessed from multiple commands or parts of the program?

## When should I use `Arc<Mutex<T>>`?
- If you need to access the value somewhere not managed by Tauri.