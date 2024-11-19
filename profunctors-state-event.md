# Profunctors and State->Event

Profunctor's most common use in programming is as a composition scheme that abstracts pre/post routine.

`let dimap = (f, g) => (h, a) => g(h(f(a)))`

The dimap reorganises functions f and g in such a way that f runs on [a] before it reaches the middle function h and then the results of h is plugged as an input to g.

### The easiest to understand example: Working on a string

It's unfortunate that we don't have a string as syntactic sugar over list of characters in JS as it is it's own wonky object so whenever we have to do something more involved with a string, the pre-post routine is to convert it into an array and back to string.

```
let overString = dimap(
  str => str.split(''), // str to arr
  arr => arr.join('') // arr to str
)

console.log(
  overString(
    listChar => listChar.map(c => c+'-'),
    'foobar'
  )
)

// -> f-o-o-b-a-r-
```

### Another example is get and "set"

```
let get = k => o => o[k];
let set = k => v => ({ [k]: v });

let overFoo = dimap(
  get('foo'),
  set('foo')
);

let state = {
  foo: 12,
  bar: 21
};

console.log(
  overFoo(
    n => n+1,
    state
  )
)

// -> { foo: 13 }

```

In here the function f gets the value out of an object, the function h in the middle increments it, and the function "set" creates a new object with minimal representation of what changed from the original.

### Currying dimap a little deeper to separate h and a

In this form the dimap can become recursive on the function h for deeper nested objects

```
let dimap = (f, g) => h => a => g(h(f(a)))

let get = k => o => o[k];
let set = k => v => ({ [k]: v });

let state = {
  foo: {
    baz: 32
  },
  bar: 21
};


let overFoo = dimap(
  get('foo'),
  set('foo')
);

let overBaz = dimap(
  get('baz'),
  set('baz')
);

console.log(
  overFoo(
    overBaz(n => n+1)
  )(state)
)

// ->> { foo: { baz: 33 } }
```

### On composition

Since we can create any kind of getter/setter or pre/post routine that is recursive at h, we can begin to compose different kinds of dimaps, like

```
let map = f => F => F.map(f);

console.log(
  overFoo(
    overString(
      map(c => c+'-')
    )
  )
)

// -> { foo: 'f-o-o-b-a-r-' }
```

Since the function h in the middle is just any function, we can pass overBaz as an input and continue deeper in.

### Last example before diving into events, null checking with a set of objects

```
let dimap = (f, g) => h => a => g(h(f(a)))
let map = f => F => F.map(f);
let fold = f => F => F.fold(f);

let Id = a => ({
  type: 'Id',
  a,
  map: f => Id(f(a)),
  fold: f => f(a),
  chain: f => f(a)
})

let None = a => ({
  type: 'None',
  a,
  map: f => None(a),
  fold: f => a,
  chain: f => None(a)
})

let Optional = a => a ? Id(a) : None('Value didnt exist');

let overOptional = dimap(
  Optional,
  fold(a => a)
)

console.log(
  'Case without a value: ',
  overOptional(
    map(n => n+1)
  )(null),
)
// -> 'Value didnt exist'

console.log(
  'Case with a value: ',
  overOptional(
    map(n => n+1)
  )(12),
)
// -> 13
```
In here we use an Optional construction as the pre-routine to bind the input [a] value in a set of two objects that have indentical set of methods in their interface. In the case of Id object, the space has a point for the h to run, in the case of None, it doesn't. The post routine is to extract the value from this optional context. Out comes an error message or value that was transformed via the h.

This syntax is expensive for None, but it can just be a traditional object that returns 'this' instead of creating a new one. But that's besides the point.

### On timestamps and events

The next thing is to create timestamps for these State->Event style dimaps going back to the get/set example

```
let set = k => v => ({
  [k]: v,
  timestamp: new Date().toISOString(),
  author: getUser()
});

let overFoo = dimap(get('foo), set('foo'))

let state = {
  foo: 12,
  bar: 21
}

console.log(
  overFoo(n => n+1)(state)
)
// -> { foo: 13, timestamp: '2015-11-19T11:02:50.904Z', author: 'Jake' }
```

This gives us a transaction log of what changed in our state, recursively with the minimal representation in however deep nested set of data that we can then use to reconstruct the state based on the most recently cached state.

### Putting it together

```
let dimap = (f, g) => h => a => g(h(f(a)))
let map = f => F => F.map(f);
let fold = f => F => F.fold(f);

let overString = dimap(
  str => str.split(''),
  arr => arr.join('')
)

let get = k => o => o[k];
let set = k => v => ({
  [k]: v,
  timestamp: new Date().toISOString(),
  user: getUser()
});

let state1 = {
  foo: {
    bar: 'qweasd'
  }
};

let state2 = {
  foo: {
    bar: null
  }
}

let overFoo = dimap(get('foo'), set('foo'));
let overBar = dimap(get('bar'), set('bar'));

let Id = a => ({
  type: 'Id',
  a,
  map: f => Id(f(a)),
  fold: f => f(a),
  chain: f => f(a)
})

let None = a => ({
  type: 'None',
  a,
  map: f => None(a),
  fold: f => a,
  chain: f => None(a)
})

let Optional = a => a ? Id(a) : None('Value didnt exist');

let overOptional = dimap(
  Optional,
  fold(a => a)
)

console.log(
  'success case',
  overFoo(
    overBar(
      overOptional(
        map(
          overString(
            map(c => c + '-')
          )
        )
      )
    )
  )(state1)
)

// { foo: { bar: 'q-w-e-a-s-d-', timestamp: '...', author: 'Jake' }, timestamp: '...',  author: 'Jake' }


console.log(
  'error case',
  overFoo(
    overBar(
      overOptional(
        map(
          overString(
            map(c => c + '-')
          )
        )
      )
    )
  )(state2)
)
// -> { foo: { bar: 'Value didnt exist', timestamp: '...', author: 'Jake' }, timestamp: '...', author: 'Jake' }
```

From here on out it's just about denesting the syntax with a piping function `let pipe = (...fs) => fs.reduce((f, g) => a => g(f(a)))` and sending these events as payload to some reconsiliation function that merges the list of events with the state, sorted by timestamps.