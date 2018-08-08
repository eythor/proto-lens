# proto-lens-tutorial

## Table of Contents

1. [Message Generation](#message-generation)
2. [Oneof Generation](#oneof-generation)
3. [Enum Generation](#enum-generation)
4. [Field Overloading](#field-overloading)
5. [Any](#any)
6. [Repeated](#repeated)
7. [Map](#map)
8. [Lens Laws](#lens-laws)
9. [Example: Person](https://github.com/google/proto-lens/tree/master/proto-lens-tutorial/person)
10. [Example: Coffee Order](https://github.com/google/proto-lens/tree/master/proto-lens-tutorial/coffee-order)

## Message Generation

`message`s that are defined in a `.proto` file are generated as Haskell records. Given instances to various typeclasses for making their use more ergonomic in code use.

A `message` may be defined in a file `foo.proto`:
``` protobuf
syntax="proto3";

message Bar {
  int32 baz = 1;
  string bippy = 2;
}
```

This will generate a `Foo` module with the a `Bar` record containing fields `_Bar'baz` and `_Bar'bippy`:
``` haskell
data Bar = Bar
  { _Bar'baz   :: !Prelude.Float
  , _Bar'bippy :: !Data.Text.Text
  , _Foo'_unknownFields :: !Data.ProtoLens.FieldSet
  }
  deriving (Prelude.Show, Prelude.Eq, Prelude.Ord)
```

Notice `_Foo'_unknownFields :: !Data.ProtoLens.FieldSet`; it stores fields that are not recognized during deserialization (for example, if the message was generated by a newer version of the `.proto` file), those fields will be included if the message is later serialized.

Instances generated are:

* `Lens.Labels.HasLens` and `Lens.Labels.HasLens'` are for overloading field names (see [Field Overloading](#field-overloading).
* `Data.Default.Class.Default` for having [default message values](https://developers.google.com/protocol-buffers/docs/proto3#default).
* `Data.ProtoLens.Message` for enabling serialization by providing reflection of all of the fields that may be used by this type.

## Oneof Generation

A `oneof` group is generated as a field in the record, where the field is a `Maybe` in case none of the cases are present (or a case is present from a later version of the proto).

A `message` with a `oneof` may be defined in a file `foo.proto`:
``` protobuf
syntax="proto3";

message Foo {
  oneof bar {
    int32 baz = 1;
    string bippy = 2;
  }
}
```

This will generate a `Foo` module with the a `Bar` record containing the field `_Foo'bar` and a coproduct `Foo'Bar` with constructors `Foo'Baz` and `Foo'Bippy`. On top of this, `Prism'` functions will be generated for the sum type, in this case `Foo'Bar`, one for each `oneof` field:
``` haskell
data Foo = Foo
  { _Foo'bar :: !(Prelude.Maybe Foo'Bar)
  , _Foo'_unknownFields :: !Data.ProtoLens.FieldSet
  }
  deriving (Prelude.Show, Prelude.Eq, Prelude.Ord)

data Foo'Bar = Foo'Baz !Data.Int.Int32
             | Foo'Bippy !Data.Text.Text
             deriving (Prelude.Show, Prelude.Eq, Prelude.Ord)

_Foo'Baz :: Lens.Labels.Prism.Prism' Foo'Bar Data.Int.Int32
_Foo'Baz
 = Lens.Labels.Prism.prism' Foo'Baz
     (\ p__ ->
        case p__ of
            Foo'Baz p__val -> Prelude.Just p__val
            _otherwise -> Prelude.Nothing)

_Foo'Bippy :: Lens.Labels.Prism.Prism' Foo'Bar Data.Text.Text
_Foo'Bippy
 = Lens.Labels.Prism.prism' Foo'Bippy
     (\ p__ ->
        case p__ of
            Foo'Bippy p__val -> Prelude.Just p__val
            _otherwise -> Prelude.Nothing)
```

The `Prism'` functions allow us to succinctly focus on one branch of the sum type for our Message, for example:
``` haskell
import Lens.Labels.Prism

accessBaz :: Foo -> Maybe Int32
accessBaz foo = foo
             ^? maybe'bar -- We want to look at the 'bar' oneof field
              . _Just     -- We only care if this value is set with a `Just`
              . _Foo'Baz  -- Focus on the 'baz' branch of our sum type

-- | Creates a 'Foo' with an incoming 'Int32'
createFoo :: Int32 -> Foo
createFoo i = def & maybe'bar .~ (_Just # _Foo'Baz # i)

-- | Sets a new `bippy` value
updateFoo :: String -> Foo -> Foo
updateFoo s foo = foo ?~ _Foo'Bippy # s
```

Our [previously mentioned instances](#message-generation) are generated but we will note the following about `HasLens'`:

* `Lens.Labels.HasLens'` also include `maybe'*` `HasLens'` instances for viewing the individual cases as `Maybe` values.

## Enum Generation

`enum`s that are defined in a `.proto` file are generated as Haskell coproducts. Given instances to various typeclasses for making their use more ergonomic in code use.

An `enum` may be defined in a file `foo.proto`:
``` protobuf
syntax="proto3";

enum Baz {
  BAZ1 = 0;
  BAZ2 = 1;
}
```

This will generate a `Foo` module with the a `Bar` coproduct containing three constructors:
``` haskell
data Baz = BAZ1
         | BAZ2
         | Baz'Unrecognized !Baz'UnrecognizedValue
         deriving (Prelude.Show, Prelude.Eq, Prelude.Ord)
```

The `Bar'Unrecognized` constructor will be created during deserialization if the `enum` field is set to a numeric value that is not listed for that `enum` in the `.proto` file. For example, given the case above, the user calls `decodeProto` which encounters the numeric value `2` the value set for `Baz` would be `Baz'Unrecognized (Baz'UnrecognizedValue 2)`.

When using `proto2` syntax there are a few notes to remember when using `enum` data:
  * The default is `minBound` unless an [explicit default value](https://developers.google.com/protocol-buffers/docs/proto#optional) is given.
  * `Bar'Unrecognized` would not be generated.

When using `proto3` syntax it is important to rememver that the first `enum` value must be zero.

Instances generated are:

* `Data.ProtoLens.MessageEnum` for enabling safe decoding.
* `Prelude.Bounded` where `maxBound` is the maximum numbered field, and `minBound` is the minimum.
* `Prelude.Enum` where the numbering of the fields dictates the enumeration.
* `Data.Default.Class.Default` where the default value is `minBound`.
* `Data.ProtoLens.FieldDefault` same as `Default`.

## Field Overloading

When we look at having the `message`:
``` protobuf
syntax="proto3";

message Bar {
  int32 baz = 1;
  string bippy = 2;
}
```
we said that `baz` and `bippy` accessors are created via `HasLens'` instances. If we add a further message into the mix such as:
``` protobuf
message Foo {
  string baz = 1;
}
```
we can see that `baz` is common to both `Bar` and `Foo`. The difference will be that the instances for `HasLens'` will be:
``` haskell
instance Prelude.Functor f =>
         Lens.Labels.HasLens' f Foo "baz" (Data.Text.Text)

instance Prelude.Functor f =>
        Lens.Labels.HasLens' f Bar "baz" (Data.Int.Int32)
```
The fields are overloaded on the symbol `baz` but connect `Foo` to `Text` and `Bar` to `Int32`. Then we can find that there is one, polymorphic definition in the `Foo_Fields.hs` file:
``` haskell
baz ::
    forall f s t a b . (Lens.Labels.HasLens f s t "baz" a b) =>
      Lens.Family2.LensLike f s t a b
baz
  = Lens.Labels.lensOf
      ((Lens.Labels.proxy#) :: (Lens.Labels.Proxy#) "baz")
```
If we have any other records that also contain `baz` from other modules these lenses could also be used to access them. We should take care in these cases as to only import one version of `baz` when we are doing this, otherwise name clashes will occur.

The use of `baz` can be done in two ways and which way you choose is up to you and your style. The first is by importing the `*_Fields.hs` module, for example:
``` haskell
import Microlens           ((^.))
import Proto.Foo        as P
import Proto.Foo_Fields as P

myBar :: P.Bar
myBar = def & P.baz   .~ 42
            & P.bippy .~ "querty"

main :: IO ()
main = putStrLn $ myBar ^. P.bippy
```

The second method is by using the `OverloadedLabels` extension and importing the orphan instance of `IsLabel` for `proto-lens` `LensFn` type, giving us the use of `#` for prefixing our field accessors. To bring this instance into scope we need to also import `Lens.Labels.Unwrapped`:
``` haskell
{-# LANGUAGE OverloadedLabels #-}

import Lens.Labels.Unwrapped ()
import Microlens             ((^.))
import Proto.Foo          as P

myBar :: P.Bar
myBar = def & #baz   .~ 42
            & #bippy .~ "querty"

main :: IO ()
main = putStrLn $ myBar ^. #bippy
```

## Any

An `Any` field stands for any arbitrary message and thus is represented by an arbitrary blob of bytes. We can see this as a placeholder for any user defined message where the message becomes concrete when we unpack it to some message we have chosen. There are two utility functions for packing any `Message a` into an `Any` and its dual for unpacking an `Any` into a `Message a`. These functions are called `pack` and `unpack` rsepectively and their type signatures are below:

``` haskell
pack :: forall a. Message a => a -> Any
unpack :: forall a. Message a => Any -> Either UnpackError a
```

The `Any` type and its utility functions are provided by the `Data.ProtoLens.Any` module in the `proto-lens-protobuf-types` package.

Further information on `Any` and how it works in the protocol can found in the [official documentation](https://developers.google.com/protocol-buffers/docs/proto3#any)

## Repeated

`repeated` fields signify that the type of the field is a list of values, narturally fitting to the `[a]` type in Haskell. For example:
``` protobuf
message Foo {
  repeated int32 a = 1;
  repeated int32 b = 2 [packed=true];
}
```
generates:
``` haskell
data Foo = Foo
  { _Foo'a :: ![Data.Int.Int32]
  , _Foo'b :: ![Data.Int.Int32]
  , _Foo'_unknownFields :: !Data.ProtoLens.FieldSet
  }
  deriving (Prelude.Show, Prelude.Eq, Prelude.Ord)
```

## Map

`map` fields signify that the type of the field is mapping from one value to another, narturally fitting to the `Data.Map a b` type in Haskell. For exmaple:
``` protobuf
message Foo {
  map<int32, string> bar = 1;
}
```
generates:
``` haskell
data Foo = Foo
  { _Foo'bar :: !(Data.Map.Map Data.Int.Int32 Data.Text.Text)
  , _Foo'_unknownFields :: !Data.ProtoLens.FieldSet
  }
  deriving (Prelude.Show, Prelude.Eq, Prelude.Ord)

data Foo'BarEntry = Foo'BarEntry
  { _Foo'BarEntry'key :: !Data.Int.Int32
  , _Foo'BarEntry'value :: !Data.Text.Text
  , _Foo'BarEntry'_unknownFields :: !Data.ProtoLens.FieldSet
  }
  deriving (Prelude.Show, Prelude.Eq, Prelude.Ord)
```

`Foo'BarEntry` is generated due to [backwards compatability](https://developers.google.com/protocol-buffers/docs/proto3#maps), so we can ignore this generated code and focus on the fact that we can treat this data as a regular Haskell `Map`.

## Encode/Decode and Show/Read

`Data.ProtoLens` provides utilities for converting to and from `ByteString` and `Text` values. For `ByteString` we are provided the `encodeMessage` and `decodeMessage` functions. For `Text` we are provided the `showMessage` and `readMessage` functions. The former are used for encoding and decoding to/from wire format. While the latter are used converting and parsing human readable representations. The type signatures for these functions are given below:

``` haskell
encodeMessage :: Message msg => msg        -> ByteString
decodeMessage :: Message msg => ByteString -> Either String msg

showMessage :: Message msg => msg  -> String
readMessage :: Message msg => Text -> Either String msg
```

## Lens Laws

Underneath there is a function that is used for creating lenses:
``` haskell
Data.ProtoLens.maybeLens :: a -> Lens' (Maybe a) a
```

We should note that `maybeLens` does not satisfy the lens laws, which expect that:
``` haskell
set l (view l x) == x
```

An example of an offending case is:
``` haskell
set (maybeLens 'a') (view (maybeLens 'a') Nothing) == Just 'a'
```

However, this is the behavior generally expected by users, and only matters if we're explicitly checking whether a field is set.

Another pitfall is when interacting with `oneof` fields it is possible to clear existing values. For example if we have the following proto:

``` protobuf
message Foo {
  oneof bar {
    int32 baz = 1;
    string bippy = 2;
  }
}
```
we can end up doing the following:
``` haskell
fooVal :: P.Foo
fooVal = def & P.maybe'baz ?~ 42

fooVal' :: P.Foo
fooVal' = fooVal & P.maybe'bippy .~ Nothing

main :: IO ()
main = do
  print fooVal  -- outputs: Foo {_Foo'bar = Just (Foo'Baz 42), _Foo'_unknownFields = []}
  print fooVal' -- outputs: Foo {_Foo'bar = Nothing, _Foo'_unknownFields = []}
```
We have cleared the previously set `Just (Foo'Baz 42)` value by doing `P.maybe'bippy .~ Nothing`. To try and avoid this it would be best to organise your code by using the `Prism'` functions for `oneof` fields instead.