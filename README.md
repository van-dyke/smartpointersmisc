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
