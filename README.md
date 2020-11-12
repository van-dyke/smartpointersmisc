# 1. enable_shared_from_this explanation 
(taken from: https://stackoverflow.com/questions/712279/what-is-the-usefulness-of-enable-shared-from-this )

***enable_shared_from_this*** adds a private ***weak_ptr*** instance to ***T*** which holds the 'one true reference count' for the instance of T.

So, when you first create a ***shared_ptr*** onto a new T*, that T*'s internal weak_ptr gets initialized with a refcount of 1. ***The new shared_ptr basically backs onto this weak_ptr.***

T can then, in its methods, call shared_from_this to obtain an instance of shared_ptr<T> that backs onto the same internally stored reference count. This way, you always have one place where T*'s ref-count is stored rather than having multiple shared_ptr instances that don't know about each other, and each think they are the shared_ptr that is in charge of ref-counting T and deleting it when their ref-count reaches zero.


It is to enable the ability to return this as a shared pointer since this gives you a raw pointer.

in other word, it allows you to turn code like this:
```cpp
class Node {
public:
    Node* getParent const() {
        if (m_parent) {
            return m_parent;
        } else {
            return this;
        }
    }

private:

    Node * m_parent = nullptr;
};           
```

into this:

```cpp
class Node : std::enable_shared_from_this<Node> {
public:
    std::shared_ptr<Node> getParent const() {
        std::shared_ptr<Node> parent = m_parent.lock();
        if (parent) {
            return parent;
        } else {
            return shared_from_this();
        }
    }

private:

    std::weak_ptr<Node> m_parent;
};           
```

Code like this won't work correctly:
```cpp
int *ip = new int;
shared_ptr<int> sp1(ip);
shared_ptr<int> sp2(ip);
```

Neither of the two shared_ptr objects knows about the other, so both will try to release the resource when they are destroyed. That usually leads to problems.

Similarly, if a member function needs a shared_ptr object that owns the object that it's being called on, it can't just create an object on the fly:

```cpp
struct S
{
  shared_ptr<S> dangerous()
  {
     return shared_ptr<S>(this);   // don't do this!
  }
};

int main()
{
   shared_ptr<S> sp1(new S);
   shared_ptr<S> sp2 = sp1->dangerous();
   return 0;
}
```

This code has the same problem as the earlier example, although in a more subtle form. When it is constructed, the shared_ptr object sp1 owns the newly allocated resource. The code inside the member function S::dangerous doesn't know about that shared_ptr object, so the shared_ptr object that it returns is distinct from sp1. Copying the new shared_ptr object to sp2 doesn't help; when sp2 goes out of scope, it will release the resource, and when sp1 goes out of scope, it will release the resource again.

The way to avoid this problem is to use the class template enable_shared_from_this. The template takes one template type argument, which is the name of the class that defines the managed resource. That class must, in turn, be derived publicly from the template; like this:

```cpp
struct S : enable_shared_from_this<S>
{
  shared_ptr<S> not_dangerous()
  {
    return shared_from_this();
  }
};

int main()
{
   shared_ptr<S> sp1(new S);
   shared_ptr<S> sp2 = sp1->not_dangerous();
   return 0;
}
```

When you do this, keep in mind that the object on which you call shared_from_this must be owned by a shared_ptr object. This won't work:

```cpp
int main()
{
   S *p = new S;
   shared_ptr<S> sp2 = p->not_dangerous();     // don't do this
}
```


# 2. Using std::move on shared_ptr object argument inside the function
(taken from: https://stackoverflow.com/questions/41871115/why-would-i-stdmove-an-stdshared-ptr )

Let's suggest a better form of transfering shared_ptr object argument inside function body
```cpp
void CompilerInstance::setInvocation(
    std::shared_ptr<CompilerInvocation> Value) {
  Invocation = std::move(Value);
}
```

instead using that

```cpp
void CompilerInstance::setInvocation(
    std::shared_ptr<CompilerInvocation> Value) {
  Invocation = Value;
}
```

Move operations (like move constructor) for std::shared_ptr are cheap, as they basically are "stealing pointers" (from source to destination; to be more precise, the whole state control block is "stolen" from source to destination, including the reference count information).

Instead copy operations on std::shared_ptr invoke atomic reference count increase (i.e. not just ++RefCount on an integer RefCount data member, but e.g. calling InterlockedIncrement on Windows), which is more expensive than just stealing pointers/state.

So, analyzing the ref count dynamics of this case in details:

```cpp
// shared_ptr<CompilerInvocation> sp;
compilerInstance.setInvocation(sp);
```
If you pass ***sp*** by value and then take a copy inside the CompilerInstance::setInvocation method, you have:

1. When entering the method, the shared_ptr parameter is copy constructed: ref count atomic increment.

2. Inside the method's body, you copy the shared_ptr parameter into the data member: ref count atomic increment.

3. When exiting the method, the shared_ptr parameter is destructed: ref count atomic decrement.

You have two atomic increments and one atomic decrement, for a total of three atomic operations.

Instead, if you pass the shared_ptr parameter by value and then std::move inside the method (as properly done in Clang's code), you have:

1. When entering the method, the shared_ptr parameter is copy constructed: ref count atomic increment.

2. Inside the method's body, you std::move the shared_ptr parameter into the data member: ref count does not change! You are just stealing pointers/state: no expensive atomic ref count operations are involved.

3. When exiting the method, the shared_ptr parameter is destructed; but since you moved in step 2, there's nothing to destruct, as the shared_ptr parameter is not pointing to anything anymore. Again, no atomic decrement happens in this case.

Bottom line: in this case you get just one ref count atomic increment, i.e. just one atomic operation.
As you can see, this is much better than two atomic increments plus one atomic decrement (for a total of three atomic operations) for the copy case.

***By using move you avoid increasing, and then immediately decreasing, the number of shares. That might save you some expensive atomic operations on the use count.***
