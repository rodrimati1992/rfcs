- Feature Name: phantom_generic_attr
- Start Date: 2018-10-13
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Add a `phantomgeneric` attribute to guarantee that the listed generic parameters (type/const) can not 
change the layout of the type they are stored inside of (and any type containing that one) 
regardless of their value.

# Motivation

Say that we have a document which is deserialized,
and immediately after deserializing it we must calculate some information in the cache.

```

pub type Document<S,Init>=
    DocumentInternal<S,ConstWrapper<Init>>;

pub struct DocumentInternal<S,Init>{
    string:S,
    cache:Mutex<Cache<S>>,
    _initialization:ConstWrapper<Init>,
}

/// ConstWrapper implements more traits than PhantomData.
/// Guarantees that it is always zero-sized and has an alignment of 1.
pub struct ConstWrapper<Constant>(
    pub PhantomData<Constant>
)


pub struct Uninitialized;
pub struct Initialized;


impl<S> Document<S,Uninitialized>
where 
    S:AsRef<str>,
{
    fn initialize_inner(&self){
        self.cache.lock().expect("initialize must never panic").initialize();
    }
    pub fn initialize(mut self)->Document<S,Initialized>{
        self.initialize_inner();
        unsafe{ ::std::mem::transmute(self) }
    }
    pub fn initialize_arc(this:Arc<Self>)->Arc<Document<S,Initialized>>{
        self.initialize_inner();
        unsafe{ ::std::mem::transmute(this) }
    }
}

impl<S> Document<S,Uninitialized>
where 
    S:AsRef<str>,
{
    fn paragraph(&self,paragraph_number:usize)->Paragraph<S>{ ... }
    fn sentence(&self,paragraph_number:usize,sentence_number:usize)->Paragraph<S>{ ... }
}


```

One might notice the conversion 
from `Arc<Document<S,Uninitialized>>` to `Arc<Document<S,Uninitialized>>`
which currently works fine,since field at this moment is determined by field size,alignemnt and 
valid values for the type.

However,work in the unsafe code guidelines indicates that this is not gurateed to work 
in the future,requiring writers of unsafe code to remove any code that relies on the 
current behavior.

# Reference-level explanation

The attribute `phantomgeneric` guarantees that any generic parameter it lists 
cannot affect the layout of the datatype that declared it or 
the layout of types storing the declared type.



### Syntax

#[ phantomgeneric ( ( <generic_parameter_ident> ),*  ) ]

`<generic_parameter_ident>` is the identifier of a generic parameter (type/const) of 
this attribute is applied to.

### Preconditions

This attribute would require that fields mentioning generic parameters (type and const) 
to be types with the same attribute,or PhantomData .

### Postconditions

The declared type won't change its layout when the phantom generic parameters change.

Any datatype storing the type which declares phantom generic parameters will not
change its layout when the phantom generic parameters change,
unless they go through a trait 
(ie:store <DeclaredType<PhantomGeneric> as Iterator>::Item).


## Interaction with `#[repr(...)]` attribute

This attribute is entirely orthogonal to representation attributes,since it only 
affects generic parameters which are stored in a PhantomData/types with the same attribute.

### Examples

We will these pseudocode items:

- the `eq_layout` function to determine which memory layouts are compatible.
- the `assert_eq_layout` macro to assert that the memory layout must be the same.
- the `assert_ne_layout` macro to assert that the memory layout 
    is not guaranteed to be the same by `phantomgeneric`.

#### Example 1, allowed

This example demonstrates a type with a phantom type parameter:

```

pub type Document<S,Init>=
    DocumentInternal<S,ConstWrapper<Init>>;

#[phantomgeneric(Init)]
pub struct DocumentInternal<S,Init>{
    string:S,
    cache:Mutex<Cache<S>>,
    _initialization:ConstWrapper<Init>,
}

#[phantomgeneric(Init)]
pub struct ConstWrapper<Constant>(
    pub PhantomData<Constant>
)

pub struct Uninitialized;
pub struct Initialized;


assert_eq_layout!( 
    Document<Box<str> , Uninitialized>,
    Document<Box<str> , Initialized >
);
assert_eq_layout!( 
    Document<String , Uninitialized>,
    Document<String , Initialized >
);
assert_eq_layout!( 
    Arc<Document<String , Uninitialized>>,
    Arc<Document<String , Initialized >>
);
assert_eq_layout!( 
    Box<Document<String , Uninitialized>>,
    Box<Document<String , Initialized >>
);
assert_eq_layout!( 
    &Document<String , Uninitialized>,
    &Document<String , Initialized >
);
assert_eq_layout!( 
    &mut Document<String , Uninitialized>,
    &mut Document<String , Initialized >
);

```

### Example 2

This demonstrates that if the `phantomgeneric` 
attribute is not used the memory layout is not guaranteed:
```

#[repr(C)]
pub StrWrapper<T,I>(T,PhantomData<I>)

assert_ne_layout!( 
    StrWrapper<String,Uninitialized> ,
    StrWrapper<String,Initialized  > 
);

```

To make it the same memory layout,add the `#[phantomgeneric(I)]` attribute to StrWrapper.
```

#[phantomgeneric(I)]
#[repr(C)]
pub StrWrapper<T,I>(T,PhantomData<I>)

assert_eq_layout!( 
    StrWrapper<String,Uninitialized>,
    StrWrapper<String,Initialized  >
);

```



#### Example 3, allowed , const-generics

The `phantomgeneric` attribute is compatible with const generic,this is an example of using it:

```
#[derive(Debug,Copy,Clone,PartialEq,Eq)]
enum Initialization{
    Uninitialized,
    Initialized,
}
use self::Initialization::*;

#[phantomgeneric(INIT)]
#[repr(C)]
pub struct Document<S,const INIT:Initialization >{
    string:S,
    cache:Mutex<Cache<S>>,
}   

assert_eq_layout!( 
    Document<Box<str> , Uninitialized>,
    Document<Box<str> , Initialized >
);
assert_eq_layout!( 
    Document<String , Uninitialized>,
    Document<String , Initialized >
);
assert_eq_layout!( 
    Arc<Document<String , Uninitialized>>,
    Arc<Document<String , Initialized >>
);
assert_eq_layout!( 
    Box<Document<String , Uninitialized>>,
    Box<Document<String , Initialized >>
);
assert_eq_layout!( 
    &Document<String , Uninitialized>,
    &Document<String , Initialized >
);
assert_eq_layout!( 
    &mut Document<String , Uninitialized>,
    &mut Document<String , Initialized >
);


```

#### Example 4 ,disallowed

This example is forbidden,since it involves storing a type which takes the 
phantom generic parameter as an input:

```
#[phantomgeneric(I)]
struct WithItem<I>
where 
    I:Iterator,
{
    item:I::Item
}

```

#### Example 5 ,allowed

This example is allowed,since `I` is used as a generic parameter to a type that 
uses `phantomgeneric` for that parameter: 

```
#[phantomgeneric(I)]
struct WithItem<I>
where 
    I:Iterator,
{
    item:Wrapper<u32,u32,I>,
}

#[phantomgeneric(Constant)]
struct Wrapper<T,U,I>{
    value0:T,
    value1:U,
    value2:PhantomData<I>,
}


```


# Guide level explanation 

Include this in the nomicon about representation attributes:

```
If one wants to add a phantom generic parameter,as in a generic parameter which does not get instantiated and is not part of any instantiated type as regular generic parameter ,
one can use the `phantomgeneric` attribute to guarantee that the generic parameter cannot affect 
the layout of the type or any type that contains it.

### Example 

    #[phantomgeneric(C)]
    struct WithMetadata<T,U,C>{
        value0:T,
        value1:U,
        _metadata:PhantomData<C>,
    }

    fn change_metadata<T,U,C0,C1>( meta:WithMetadata<T,U,C0> )->WithMetadata<T,U,C1> {
        unsafe{ ::std::mem::transmute(meta) }
    }

    fn change_metadata_arc<T,U,C0,C1>( meta:Arc<WithMetadata<T,U,C0>>)->Arc<WithMetadata<T,U,C1>>{
        unsafe{ ::std::mem::transmute(meta) }
    }

    struct Happy;
    struct Grumpy;

    let with_metadata: WithMetadata<_,_,Happy>=Arc::new(WithMetadata{
        value0:"hello world!",
        value1:(),
        _metadata:Default::default(),
    });

    let with_metadata: WithMetadata<_,_,Grumpy> =change_metadata_arc(with_metadata);


```



# Drawbacks 

This attribute interferes with any compiler plugin which could reorder fields of 
a type when a generic parameter changes,

The compiler plugins this would affect does not include:
- reordering fields in the source code .
- reordering fields without regard for the generic parameters.


# Rationale

Allowing unsafe code to make assumptions about the layout of specific types,
without having to choose a specific `#[repr(..)]` attributes,
since those attributes have drawback that are not necessary for adding
phantom generic parameters.
The `repr(C)` attribute does not provide any layout optimizations and does not 
prevent users of the type from changing their layout when the phantom generic parameter changes.


# Alternatives

Make the layout of types only depend on field type size/alignment/illegal-values.

### #[repr(as(..))]

Provide a similar attribute `#[repr(as( C="()" ))]` which treats generic parameters
as though they have the layout of the value assigned.
Examples:
```
#[repr(as( C="()" ))]
struct WithMeta<T,C>{
    value:T,
    _marker:PhantomData<C>,
}

```
The type parameter `C` is treated as the `()` type for the purposes of layout.


```

#[repr(as( VALUE=true ))]
struct WithMeta<T,const VALUE:bool>{
    value:T,
}


```
The const parameter `VALUE` is treated as the `false` value for the purposes of layout.

### Create another RFC with a different design

Do nothing is not an option the author of this RFC will settle for.


# Prior Art.

None that the author is aware.
