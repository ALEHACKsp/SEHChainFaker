###### Warning! This code is not perfect and has its imperfections. It is meant to be shown as a PoC, not a fully working bypass.

***

**Code**

```cpp
void fakeChain(DWORD* chain)
{
	chain[1] = 0x1555555;
	((DWORD*)chain[0])[1] = 0x1555555;
}

void restoreChain(DWORD* chain, DWORD unk, DWORD unk1)
{
	chain[1] = unk;
	((DWORD*)chain[0])[1] = unk1;
}
```

*Alternative Styling*

```cpp
#define fakeExceptionChain(f)   [&]() { \
                    DWORD* __exceptionChain = (DWORD*)__readfsdword(0);                                 \
                    DWORD  __unk = __exceptionChain[1], __nextunk = ((DWORD*)__exceptionChain[0])[1];   \
                    __exceptionChain[1] = 0x1555555;                                                    \
                    ((DWORD*)__exceptionChain[0])[1] = 0x1555555;                                       \
                    auto __ret = (f);                                                                   \
                    __exceptionChain[1] = __unk;                                                        \
                    ((DWORD*)__exceptionChain[0])[1] = __nextunk;                                       \
                    return __ret;                                                                       \
                }()
```

***

**Info**

This bypass works off a simple observation made after looking at luaD_rawrunprotected, which I am going to explain now.

At first glance it may look confusing, but with the help of clever commenting we can reconstruct a better view of everything important going on.
![Alt text](https://vgy.me/Uy14BW.png "Decompiled view")

If you keep looking in that loop, you can see that after two successful bounds checks it will jump to the good ol' function address check (which can be bypassed using other publicly available methods, such as my own) and then it will call the closure.

This function becomes much more simple to read if it is wrote as pseudo code. (with the help of the Lua source)

(Code from Lua will me marked with two asterisks)
```
lua_longjump lj; **
exceptionChain = fs[0/18h];
handlerDepth = 1;

lk.status = 0; **
lj.previous = rL->errorJmp; **
rL->errorJmp = &lj; **

while (exceptionChain[0] != 0xFFFFFFFF)
	if (!isInBounds(exceptionChain[1]))
		setupShutdown();
	exceptionChain = exceptionChain[0];
	if (handlerDepth++ > 2)
		goto executeClosure;

if (!isInBounds(exceptionChain[6]))
	setupShutdown();
else
	goto executeClosure;
	
executeClosure:
	f(rL, ud);

rL->errorJmp = lj.previous; **
return lj.status; **
```

And even more simple if all Lua related code is stripped.

```
exceptionChain = fs[0/18h];
handlerDepth = 1;

while (exceptionChain[0] != 0xFFFFFFFF)
	if (!isInBounds(exceptionChain[1]))
		setupShutdown();
	exceptionChain = exceptionChain[0];
	if (handlerDepth++ > 2)
		goto executeClosure;
  
if (!isInBounds(exceptionChain[6]))
	setupShutdown();
else
	goto executeClosure;
	
executeClosure:
	f(rL, ud);
```

Now we have a nice small chunk of code.
As I'm sure you can already tell, there are multiple simple ways to bypass this.

The first being:
```
exit while loop as quickly as possible (skipping any checks on handlers down the chain)
pass final bounds check
pass the closure bounds check
```
Which works by setting the next handler to -1 (skipping the entire loop), but this causes a freezing problem that I don't understand.

The second way:
```
pass two checks in the loop
pass the closure bounds check
```
This is the way that I will be using in this guide.

We can assure ourselves that this is the correct flow that needs to take by looking at the graph view of the function.

![Alt text](https://vgy.me/Lcw7IH.png "Graph view")

Now that all of the needed information is gathered, we can start writing the bypass. This could be written in inline assembly to write it, but for this case we will use the available intrinsics.

First we need get the chain and store it for later use.

```cpp
DWORD* exceptionChain = (DWORD*)readfsdword(0);
```

Then we need to bypass the first check in the loop. (by setting it to an address <= 4299165696)

```cpp
exceptionChain[1] = 0x1555555;
```

And then bypass the second check by doing the same thing, but with the next handler in the chain.

```cpp
((DWORD*)exceptionChain[0])[1] = 0x1555555;
```

In theory this should bypass the checks that are currently in place. But it will bring up some problems later on. One is if we do not restore the chain, but this is an easy fix, which I demo in the exmaple below. The other is that on some systems for SOME reason there are no registerd handlers in the chain, making the original chain access throw a read access violation. While I do know one way to get around this, I am going to leave it up to you to see if you can figure out what is going on and how to fix it.

Now with all of our reversing out of the way, we are done. Our final bypass should look like this.

***

**Example code**

```cpp
#define RESUME_ADDRESS 0xDEADBEEF

int rlua_resume(int rL, int nargs)
{
	DWORD* exceptionChain = (DWORD*)__readfsdword(0);
	DWORD unk = exceptionChain[1], unk1 = ((DWORD*)exceptionChain[0])[1];
	fakeChain(exceptionChain);
	int ret = ((int(__cdecl*)(int, int))RESUME_ADDRESS)(rL, nargs);
	restoreChain(exceptionChain, unk, unk1);

	return ret;
}
```

*Alternative styling*

```
int rlua_resume(int rL, int nargs)
{
	return fakeExceptionChain(
		((int(__cdecl*)(int, int))RESUME_ADDRESS)(rL, nargs)
	);
}
```
