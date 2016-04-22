---
published: true
title: Proxy Primer
layout: post
---
# <center> Proxies

ECMAScript 2015, or ES6, introduces Proxy, a feature that allows us to intercept and customize many of the most basic ways that we interact with Objects. Proxies are unique in how close to internal JavaScript operations they allow us to go but, as always, with great power comes great responsibility. Let's dive in to Proxy and explore some of the things to be excited and cautious (but mostly, excited) about.

## Making Objects Talk

A Proxy is made of a **target** and a **handler**: `new Proxy(target, handler)`. The *target* is the Object whose calls we want to intercept and the *handler* optionally overrides, or **traps**, those operations. If an operation happens and a trap is defined, the handler intercepts the operation. If the handler does not define a trap for an operation, it just passes the operation to the original Object. Basic operations are things like getting a property of an object (`obj.prop`) or assigning a value to a property (`obj.prop = value`).

### Moo cow, moo

We'll start simple and report the value of a property when it is looked up. In the example below, we customize property lookup by defining a *trap*, `get(target, name)`, in the `handler` and construct a Proxy with it and the target.

```javascript
    const target = {
		duck: 'QUACK',
		cow: 'MOO',
	}	
    const handler = {
		// lookup trap
        get(target, key) {
			console.log(`target says ${target[key]}!`);
			return target[key];
        },
    }
    const talkingTarget = new Proxy(target, handler);

	// now, when we get the property of this object, it is also logged! 
	talkingTarget.cow;
	> target says MOO!
	talkingTarget.unicorn;
	> target says undefined!
```

Nice! In a way, we've overridden the standard behavior of accessing a property, one of the most common (if not most common) things we do in JavaScript. With Proxy, we are able to manipulate some of the fundamental semantics of the language, something not possible before ES6. It opens up to JavaScript coders a new type of metaprogramming, or programming which can manipulate itself.

**Woah**, *right!?* We can intercept fundamental operations before they happen!

### Traps

The *handler* has a unique definition for each of the operations it can intercept so we've got to be specific. The table below lists the ones we'll get into in this article.

| Operation | Code | Trap |
|-----------|------|------|
| <center> lookup | <center> `target.key` | <center> `get(target, key)` |
| <center> assignment | <center> `target.key = value` | <center> `set(target, key, value, receiver)`  |
| <center> existence | <center> `key in target` | <center> `has(target, key)` |
| <center> ... | <center> ... | <center> ... |

> For a complete list of intercepted operations and traps, see [MDN's concise documentation][1] or the [ECMAScript meta object protocol][2].

## We can do all sorts of things with this, so let's dive in!

### Check array for values

The  `in` operation checks for the existence of some key in an Object. In JavaScript, however, Arrays are Objects so the operation simply checks whether an index exists, not the value. As an example, `'str' in ['str'] //=> false` but `0 in ['str'] //=> true`. I am sure this is well-reasoned but it isn't immediately intuitive. Proxies can intercept `in` with the *has* trap, so let's make an augmented Array factory that will check for values instead of keys.

```javascript
	function createArray(...arr) {
		// accept an array and return a Proxy
		return new Proxy(...arr, {
			// intercept 'in' operation
			has(target, value) {
				return target.indexOf(value) !== -1
			}
		})
	}

	const supplies = createArray(['tacos', 'beer', 'lime']);

	// checks for value in an array, not the index
	if ('tacos' in supplies) {
		console.log('Wahoo!');
	}
	> Wahoo!
```

Cool, eh?

> Exercise: Use a Proxy to trap the *has* of a Tree to search the tree for an element. i.e. where `element in TreeNode` returns true when some node in the Tree with a root of TreeNode has a value of element.

### Track changes

Trapping *set*, or assignment, operations (`target.key = value`, the key is the equals sign) gives us a sneak peek at a value before it is assigned. With that information, we can do things like validate values (type-check), apply the change elsewhere (data binding) or profile the data passing through. Below, we make a function `track` that accepts an Object and returns a Proxy for the Object with a history of its properties. This would be useful for keeping analytics on how data is changing or if we had some feature like an input field and we wanted users to be able to undo.

```javascript
	function track(obj) {
		const history = {};

		// add initial property values to history
		Object.keys(obj).map(prop => history[prop] = [ obj[prop] ] )

		const trackedObj = new Proxy(obj, {
			// Intercept and track changes
			set(target, key, value, receiver) {
				key in history
					? history[key].push(value)
					: history[key] = [value];
				target[key] = value;
				return true;
			}
		});

		// Attach history via a Symbol on the tracked Object
		// A Symbol avoids accidentally overriding a property, allowing us to return only a decorated Object
		const hist = Symbol.for('history');
		trackedObj[hist] = history;

		return trackedObj;
	}

	const product = {
		name: 'Chocolate!',
		price: 9.99,
		quantity: 18,
		sell(num) {
			this.quantity -= num;
			return this.price * num;
		},
	}
	const trackedProduct = track(product);

	const hist = Symbol.for('history');
	console.log(trackedProduct[hist]);
	> { name: ['Chocolate!'], price: [9.99], quantity: [18] }

	// lower the price, sell 1, then sell 2
	trackedProduct.price = 7.99;
	trackedProduct.sell(1);
	trackedProduct.sell(2);

	console.log(trackedProduct[hist])
	> { name: ['Chocolate!'],
		price: [9.99, 7.99],
		quantity: [18, 17, 15] }
```

This implementation keeps a simple history of properties in an Array. Alternatively, changes could be recorded in a database if you wanted more persistent information or the history could be kept in a diff fashion. Note that we do not decrement quantity directly yet its changes are recorded in the history. The changes are triggered indirectly by calling the `sell` function because the `this` reference when calling `sell` is the Proxy and not the original Object.  Conveniently, this requires no extra work, is a benefit of the ever-dreaded looseness of JavaScript's `this` context, and shows that Proxy continues to work as expected in Objects and classes.

Tracking with Proxy might be useful in, for example, a web scraper that scans products on Amazon and records a product's changes over time.

> For information on Symbols, see [MDN's reference][6]

### Safely access properties

How often have you been blasted by a misspelled property name or discovered that some data wasn't shaped quite how you thought it was? As projects grow, it becomes more difficult to be consistent with naming conventions, especially when collaborating. Alerting users when they've accessed an undefined piece of your library or codebase could be convenient and help avoid typos and acquaint new members of a team to an interface.

Let's use a Proxy to alert us if we try to access a property that does not exist and suggest possible alternatives.

```javascript
	// We'll use the nice and tiny fuzzy-search npm package, fuzzy,
	// for generating a list of alternative properties.
	import { fuzzy } from 'fuzzy';

	const accessCheck = {
		get(target, key) {
			if (!(key in target)) {
				const allKeys = Object.keys(target);

				// Filter all properties for fuzzy matches with the attempted key.
				const matches = fuzzy.filter(key, allKeys);

				// If there are no matches, get all keys
				// otherwise, take the 'string' of the match
				const alternatives = matches.length
					? matches.map(match => match.string)
					: allKeys

				throw new ReferenceError(
					`Property ${key} does not exist. Perhaps you meant: ${alternatives}`
				);
			}

			return target[key];
		},
	}
	const Danny = {
		user_id: 21,
		name: 'Danny',
		username: 'the bear',
	}

	const smartUser = new Proxy(Danny, accessCheck);

	console.log(smartUser.usr);
	> ReferenceError: Property usr does not exist. Perhaps you meant one of: user_id, username
	
	console.log(smartUser.user_name);
	> ReferenceError: Property user_name does not exist. Perhaps you meant one of: user_id, name, username 

	console.log(smartUser.ttt)
	> ReferenceError: Property ttt does not exist. Perhaps you meant one of: user_id, name, username.

	console.log(`Hey ${smartUser.name}!`);
	> Hey Danny!
	
```

Now, when someone tries to access a property through this Proxy, we intercept the operation with a *get* trap. It is then verified and either passes right on through or we alert the coder (via an Error) that they have a mistake and show them some potential fixes. This blows my mind. I am not 100% on its usefulness, but I love that we can add this layer on top of plain, standard, vanilla JavaScript.

This example also highlights a drawback of Proxy: unintended consequences. Because Proxy, by its nature, augments some of the most basic features of the language, problems can arise where they aren't expected. For example, I frequently check that a property exists before using it by accessing the property, i.e: `if (data.user && data.user.isRegistered)` This is a very common (and good!) pattern but would be thrown off by our Proxy. In this case, we would throw an Error if there was no `user` property in `data`, which is not what the programmer intended.

> A workaround: `if ('user' in data && data.user.isRegistered)`

To use this example more fluidly in your code, simply turn it into a function that accepts an object and applies a handler.

```javascript
	// using accessCheck from above
	function smartAccess(obj) {
		return new Proxy(obj, accessCheck);
	}
	
	fetch('user from server')
		.then(resp => resp.json())
		.then(smartAccess)
		.then(user => console.log(`Welcome, ${user.userId}!`) );
		> ReferenceError: Property userId does not exist. Perhaps you meant: user_id
```

## Object Middleware

This pattern of intercepting, doing something, and then forwarding is none other than the pervasive concept of middleware (think Express.js' `app.use(middleware)`). I'm sure it has other definitions but to me, Middleware is a layer of behavior between the beginning and end of some action. It is a powerful pattern for augmenting standard functionality, especially for providing a consistent interface for others to interact with a process. Proxies give us access to a part of JavaScript's internal pipeline so we can inject our own logic.

Let's make a small and simple interface for applying functions to *set* operations. Basically, a run a list of callbacks each time a property changes. Our interface will accept a target to Proxy, and an options Object where keys are the properties to operate on and values are a list of functions, or middleware, to apply to *set* operations for those keys. On *trapping* a *set*, we will sequentially apply the middleware, passing the return value of each function into the next middleware function. This is essentially `pipe`ing functions together or `reduce`ing the assignment's value over a list of functions.

Basic usage:

```javascript
	options = {
		key: [middleware1, middleware2, middleware3
	}
	const target = watch({ key: 123 }, options); 
	target.key = value //=> target.key = middleware_3(middleware_2(middleware_1(value)))
```

Now let's write it!

```javascript
	function watch(obj, options) {
		// Initial application of middleware 
		for (const key in options) {
		    // A null at the 0 index indicates an optional property
			// i.e. do not run middleware if property does not exist.
		    if (!(key in obj) && options[key][0] === null) continue;

			options[key].forEach(mWare => mWare(obj[key], key, obj) )
		}

		return new Proxy(obj, {
			set(target, key, value) {
				let newVal = value;
				if (key in options) {
				    // Skip null
				    let mWares = options[key][0] === null
				        ? options[key].slice(1)
				        : options[key];
	
					// Reduce middleware for this key, pass the value, key, and target
					newVal = mWares.reduce((val, mWare) => mWare(val, key, target), newVal);
				}
				target[key] = newVal;
				return true;
			}
		});
	}
```

Surprisingly simple!.. am I missing something? Now we can build a list of functions that we want to process before assignment happens. These could be tracking changes as in our History example above, validating properties, or: anything!

I think validating properties is a useful feature. We can compose a list of functions that check for characteristics that properties need to have and throw an Error if it's not valid. As our example idea, let's say we have a page where a user can create a new Organization.

Organizations have a name and an owner, and optionally have a description and annual fee. Names must be between 3 and 25 characters, owners must be emails, descriptions are less than 300 characters, and a fee must be an integer (no pesky cents). These are the constraints we'll enforce in our organization creator.

We could declare our validating middleware like this:
```javascript
	function isString(str) {
		if (typeof str !== 'string')
			throw new TypeError(`${str} is not a String.`);

		return str;
	}
```

But wait!

Luckily, we already have many a familiar and powerful library for declaring assertions such as these! Chai's assertion library throws informative errors if expectations are violated so we'll use it.

```javascript
	import { expect } from 'chai';

	function isString(str) {
		expect(str).to.be.a('string');
		return str;
	}

	function isNumber(num) {
		expect(num).to.be.a('number').and.not.equal(NaN);
		return num;
	}

	function longerThan(bound) {
		return function(test) {
			expect(test).to.have.length.above(bound);
			return test;
		}
	}

	function shorterThan(bound) {
		return function(test) {
			expect(test).to.have.length.below(bound);
			return test;
		}
	}

	function isRegistered(email) {
		// over simplified
		expect(email).to.match(/.+@.+/);
		return email;
	}

	function makeInteger(num) {
		return Number(num.toFixed());
	}
```

Each of the above functions is middleware that fits our interface. It accepts a value (and, optionally, the property name and host Object), does something with it, then returns its new representation of the value. It's important to note that order does matter: we should verify that a value is a String before checking it for String-like characteristics. To denote optional properties, pass `null` as the first middleware for a property.

```javascript
	// null at first position indicates an optional property
	const orgOptions = {
		name: [isString, shorterThan(26), longerThan(2)],
		owner: [isString, isRegistered],
		description: [null, isString, shorterThan(301)],
		fee: [null, isNumber, makeInteger],
	}
	const org = {
		name: 'Needs a name',
		owner: 'Must be registered',
	}

	const orgCreator = watch(org, orgOptions);

	orgCreator.name = 'Skydiving Scarf Knitters Association';
	> AssertionError! expected 'Skydiving Sca..' to have length below 26
	orgCreator.name = 'Skydiving Rocks!';

	// cardholding member of the skydiving scarf knitters assoc.
	orgCreator.owner = 'ferris@beul.ler';

	orgCreator.fee = 'a fine scarf';
	> AssertionError: expected 'a fine scarf' to be of type number.
	orgCreator.fee = 12.34;

	console.log(orgCreator);
	> { name: 'Skydiving Rocks!', owner: 'ferris@buel.ler', fee: 12 }
```

This implementation is pretty extensible so we could also implement tracking:

```javascript
	function recorder (val, key, target) {
		const hist = Symbol.for('history');
		const history = target[hist] = target[hist] || {};
	
		key in history
			? history[key].push(val)
			: history[key] = [val];
		
		return val;

		// Alternatively, database pseudocode
		// fetch('database$user', { method: 'PUT', body: {key: val} })
	}

	// now attach this tracker to the fee property
	orgOptions.fee.push(recorder);

	// make a new Proxy
	const orgCreator = watch(org, orgOptions);

	orgCreator.fee = 12.34;
	orgCreator.fee = 14.20;

	console.log(orgCreator[Symbol.for('history')]);
	> { fee: [12.34, 14.20] }
```

Awesome! With just another little code block we can apply history keeping middleware to any property we care about. The middlewares defined above (isString, makeInteger, etc..) can be considered plugins that could be applied elsewhere and to other Objects. This idea of pluggable Object middleware is exciting because it allows a library or community of middleware plugins. For example, a APIsubscribe plugin that allows you to subscribe an Object or property to http request like a PUT to a server/database or a GET to an API.

I am sure that this is not the best implementation of a general Object middleware, I can already see problems, but with a relatively simple piece of code, `watch(obj, options)`, we can selectively apply middleware to any property of an Object. It can be a hidden force in the background enforcing rules and structure and applying passive logic that does not have to get in the way of main application logic. Configuring middleware can reduce bloat and abstract concerns away from our main application but does considerably increase complexity.

> exercise: Allow 'watch' to accept a property on the options parameter that defines middleware that applies to the object as a whole.

> exercise: We can use this middleware to enforce interfaces! With the above change, implement middleware that validates the shape of an Object, or ensures that an Object has a certain interface (certain properties). Promises are a good example of Objects we expect to have certain properties: make a Proxy that verifies an Object is a ['thenable' (ctrl-f)][7].

## Meta end


By giving such clear access to fundamental operations like these, Proxies make it easy to perform a whole new suite of actions.

This has been a short introduction but I hope that it has shown you some of the potential of these techniques. By giving such clear access to fundamental operations, Proxies make it easy to perform a whole new suite of actions. 

For many purposes, Proxy is just another tool in the box, but often overkill, so use it wisely. If you do decide to use Proxy you may need to consider: another layer of programming logic increases time and space requirements, customizing fundamental operations can lead to unintended consequences, and, perhaps most importantly, Proxies are **not** explicit so you must be very clear when they are used.

I was first enthralled by Proxies via by Dr. Axel Rauschmayer's book, [Exploring ES6][4]. In particular, chapter 28: [Metaprogramming with proxies][5] and wholeheartedly encourage anyone with additional interest to explore his book and blog. Thanks!

## <center> I hope you learned something or, even better, had some exciting ideas!

> Reach out to me: jkrone at vt edu

[1]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy "MDN Documentation"
[2]: http://www.ecma-international.org/ecma-262/6.0/#sec-proxy-object-internal-methods-and-internal-slots "Proxy Meta Object Protocol"
[3]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Reflect "Reflect"
[4]: http://exploringjs.com/es6/ch_proxies.html "Exploring ES6"
[5]: http://exploringjs.com/es6/ch_proxies.html "Metaprogramming with proxies"
[6]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol "Symbols"
[7]: https://promisesaplus.com/ "thenable"
