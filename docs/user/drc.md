---
title:  "Using Cray DRC credentials"
---

The goal of this page is to present minimal usage of Cray Dynamic RDMA Credential (DRC)
mechanism when using mercury and libfabric. Design notes on DRC were presented
at CUG in this [paper][cug_drc]. DRC must be used on Cray systems with Aries
interconnect in order to establish cross-job communication (normally two separate
jobs are protected by *protection domains* and communication may only occur
within that domain).

[cug_drc]: https://cug.org/proceedings/cug2016_proceedings/includes/files/pap108s2-file1.pdf

## Client-Server Example

In this example we show how a server can grant access to a client job.
The first step is for the server to acquire a new DRC token by calling:

```C
uint32_t credential;

drc_acquire(&credential, 0);
```

On the client, the first step is to usually retrieve the WLM ID by calling:

```C
uint32_t wlm_id = drc_get_wlm_id();
```

The WLM ID should then be communicated to the server (using TCP for example),
which can then grant access to the client by doing:

```C
drc_grant(credential, wlm_id, DRC_FLAGS_TARGET_WLM);
```

From there, the server can simply pass around its `credential` back to the client.

Both the client and server should then use that credential to retrieve a cookie
that will then be used to communicate within the domain:

```C
drc_info_handle_t credential_info;
uint32_t cookie;

drc_access(credential, 0, &credential_info);
cookie = drc_get_first_cookie(credential_info);
```

The cookie can then be given to Mercury through the init info struct:

```C
char auth_key[16];
struct hg_init_info init_info = HG_INIT_INFO_INITIALIZER;

sprintf(auth_key, "%" PRIu32, cookie);
init_info.na_init_info.auth_key = auth_key;

hg_class = HG_Init_opt(... /* init string */, ... /* listen */, &init_info);
```

Note that eventually the DRC resources must also be released by doing:

```C
drc_release_local(&credential_info);
drc_release(credential, 0);
```