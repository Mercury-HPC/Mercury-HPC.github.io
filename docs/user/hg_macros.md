---
title: Mercury Serialization Macros
---

For convenience, Mercury provides macros that can
reduce the amount of code required to send an RPC call. Instead of using tedious
RPC stubs and code generators,
Mercury makes use of the [Boost preprocessor library](http://www.boost.org/doc/libs/release/libs/preprocessor/) so that users can generate all the boilerplate code that
is necessary to serialize and deserialize function arguments.


## RPC Registration

When registering a new RPC through the RPC layer (see
[previous](hg.md#registration) section), users are expected to tell Mercury
how to serialize and deserialize input and output arguments. To facilitate that
and in conjunction with the following macros, the `MERCURY_REGISTER()` macro
makes the registration
of RPC calls more convenient by mapping the types to the generated proc functions.

```C
MERCURY_REGISTER(hg_class, func_name, in_struct_type_name, out_struct_type_name, rpc_cb);
```

!!! example

    ```C
    int rpc_open(const char *path, rpc_handle_t handle, int *event_id);
    ```

    One can use the `MERCURY_REGISTER` macro and pass the types of the input/output
    structures directly. In cases where no input or no output argument is present,
    the `void` type can be passed to the macro.

    ```C
    rpc_id = MERCURY_REGISTER(hg_class, "rpc_open", rpc_open_in_t, rpc_open_out_t, rpc_open_cb);
    ```


### Predefined Types

Mercury already defines some types and uses standard types so that the size of
the type is fixed between platforms
when serializing and deserializing it. For convenience, HG types can also be
used to serialize bulk handles for example, but also strings, etc.

Predefined Types | Type name
---------------- | ------------
Standard types   | `int8_t`, `uint8_t` <br/> `int16_t`, `uint16_t` <br/> `int32_t`, `uint32_t` <br/> `int64_t`, `uint64_t`
Strings          | `hg_string_t`, `hg_const_string_t`
Bulk descriptor  | `hg_bulk_t`
Mercury types    | `hg_bool_t`, `hg_id_t`, `hg_size_t`, `hg_ptr_t`

## New Type Description

The macro `MERCURY_GEN_PROC()` can be used to describe new types that are
generally composed of primitive types. The macro generates both a
new struct and a proc function that can be used
to serialize arguments. The structure fields contain either input arguments or
output arguments. The generated proc routine uses the proc routines from
pre-existing types to serialize and deserialize each of the fields.

```C
MERCURY_GEN_PROC(struct_type_name, fields)
```

!!! example

    The following function has two input arguments, one output argument and one
    return value.

    ```C
    int rpc_open(const char *path, rpc_handle_t handle, int *event_id);
    ```

    The following macro can be used to generate boilerplate code for the input
    argument (again, refer to the [predefined types](#predefined-types) section for
    a list of the pre-existing types that can be passed to this macro):

    ```C
    MERCURY_GEN_PROC( rpc_open_in_t, ((hg_const_string_t)(path))
                                    ((rpc_handle_t)(handle)) )

    /* Will generate an rpc_open_in_t struct */
    typedef struct {
        hg_const_string_t path;
        rpc_handle_t handle;
    } rpc_open_in_t;

    /* and an hg_proc_rpc_open_in_t proc function */
    hg_return_t
    hg_proc_rpc_open_in_t(hg_proc_t proc, void *data)
    {
        hg_return_t ret;
        rpc_open_in_t *struct_data = (rpc_open_in_t *) data;

        ret = hg_proc_hg_const_string_t(proc, &struct_data->path);
        if (ret != HG_SUCCESS) {
          /* error */
        }
        ret = hg_proc_rpc_handle_t(proc, &struct_data->handle);
        if (ret != HG_SUCCESS) {
          /* error */
        }
        return ret;
    }
    ```

    Note the parentheses that separate the name of the field and its type. Each
    field is then separated by another pair of parentheses. This follows the sequence
    data type of the Boost preprocessor library.

## Existing `struct` Description

In some cases, however, the argument types are not known by Mercury, which is
the case of the previous example with the `rpc_handle_t` type. For these cases,
another macro, called `MERCURY_GEN_STRUCT_PROC`, can be used. It defines a
serialization function for an existing struct or type&mdash;this assumes that the
type can be mapped to already existing types; if not, users can create their
own proc function and use the `hg_proc_raw` routine that takes a stream of bytes.

```C
MERCURY_GEN_STRUCT_PROC(struct_type_name, fields)
```

!!! example

    The following function has one non-standard type, `rpc_handle_t`.

    ```C
    int rpc_open(const char *path, rpc_handle_t handle, int *event_id);

    /* pre-defined struct */
    typedef struct {
        hg_uint64_t cookie;
    } rpc_handle_t;
    ```

    The following macro can then be used to generate boilerplate code for the type
    by defining its fields.

    ```C
    MERCURY_GEN_STRUCT_PROC( rpc_handle_t, ((hg_uint64_t)(cookie)) )

    /* Will generate an hg_proc_rpc_handle_t function */
    static hg_return_t
    hg_proc_rpc_handle_t(hg_proc_t proc, void *data)
    {
        hg_return_t ret;
        rpc_handle_t *struct_data = (rpc_handle_t *) data;

        ret = hg_proc_hg_uint64_t(proc, &struct_data->cookie);
        if (ret != HG_SUCCESS) {
          /* error */
        }
        return ret;
    }
    ```
