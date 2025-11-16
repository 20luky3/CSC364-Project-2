### Bug 1 - use of unsafe function in encode_record (line 7)
- Category: unsafe function (strlen) usage
- Trigger: `rec->name` doesn't contain a null terminator, causing the program to continuously read from buffer.
- Root cause: `strlen` requires the given pointer to contain a null terminator in order to know when to stop reading from the pointer. However, since `rec->name` is a char array, not a string, it does not necessarily contain a null terminator.
- Fix: use `strnlen`, a bounded version of `strlen`. This function takes in a second argument, a limit to how much should be read from the given pointer. Even if there is no null terminator, the function won't continue to read from the buffer.

### Bug 2 - overflow of size_t in encode_record (line 8)
- Category: integer overflow
- Trigger: if `rec->score_count` is too large, it could overflow `size_t` in the calculation for space.
- Root cause: `rec->score_count` is 32 bits long, while `size_t` is typically 32 or 64 bits long (depending on the pointer size). If it is 32 bits, it is possible for the calculation of `space` to overflow due to multiplying `rec->score_count` by 2. This could result in unexpected   behaviour when using the `space` variable.
- Fix: check `rec->score_count` multiplied by 2 will be smaller than the maximum size of `size_t`. This ensures the resulting calculation will be within the size limits of `size_t`, stopping an integer overflow from occuring.

### Bug 3 - overflow if required space is too large in encode_record
- Category: buffer overflow
- Trigger: if `space` is larger than `out_cap` (the max size for `out_buf`), then a buffer overflow can occur during subsequent `memcpy` calls.
- Root cause: `space` is the number of bytes that will be written to `out_buf` in the `encode_record` function in `memcpy` calls. It is possible that `out_buf` won't have enough space to hold all the memory that will be written into it. In such cases, memory outside of `out_buf` could become corrupted and unusable.
- Fix: check `space > out_cap` before writing anything into `out_buf`. This stops any writing from occuring if `out_buf` does not have enough room to write the record in.

### Bug 4 - overflow of rec->name if name_len is too large in decode_record
- Category: buffer overflow
- Trigger: if `name_len` is larger than the size of `rec->name`, subsequent `memcpy` calls can cause a buffer overflow.
- Root cause: `rec->name` has a fixed size as defined in `mini_proto.h`. There is a `memcpy` call in `decode_record` which tries to write something into `rec->name` according to `name_len`. If `name_len` is too long, this could cause a buffer overflow and unintentionally overwrite other parts of the `rec` object.
- Fix: check that `name_len >= MAX_NAME` before the `memcpy` call. This prevents a buffer overflow from overwriting the `rec` object if `rec->name` is not large enough to hold the bytes to be written into it.

### Bug 5 - not checking if malloc worked properly in decode_record (line 41)
- Category: null pointer dereference 
- Trigger: if `rec->scores = malloc(space)` fails for any reason, usage of the pointer later in the `decode_record` function or in other callers can lead to a segmentation fault.
- Root cause: `rec->scores` is allocated memory, but it can fail for any number of reasons, and set to `NULL` as a result. The `decode_record` function doesn't check for this, so if the allocation fails, it will be trying to dereference a null pointer in future calls to `rec->name`, causing the program to crash.
- Fix: after allocating `rec->scores`, check that it is not null with `if (!rec->scores)`. This ensures the `malloc` system call did not fail before continuing to execute the rest of the function.

### Bug 6 - possible use-after-free bug after free_record
- Category: use-after-free
- Trigger: the `rec` pointer can still be called after it has been freed.
- Root cause: `free_record` frees the `rec` pointer to allow other resources to use its space. However, the address is still valid and the caller could still use it by accident. This could cause unknown behaviour since it no longer contains what the caller expects.
- Fix: set `rec->scores` and `rec` to `NULL` after they have been freed in `free_record`. This stops callers from erroneously calling the freed pointer.