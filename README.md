# Atomic ring buffer

Atomic multi-producer single-consumer ring buffer with passive tail update
and contiguous range operations.  This implementation is written in C11 and
distributed under the 2-clause BSD license.

## API

* `int ringbuf_setup(ringbuf_t *rbuf, size_t length)`
  * Setup a new ring buffer of a given _length_.  The `rbuf` is a pointer
  to the opaque ring buffer object; the caller is responsible to allocate
  the space for this object.  Typically, the object would be allocated
  dynamically if using threads or reserved in a shared memory blocked if
  using processes.  The allocation size for the object shall be obtained
  using the `ringbuf_get_sizes` function.

* `void ringbuf_get_sizes(size_t *ringbuf_size, size_t *ringbuf_local_size)`
  * Returns the size of the opaque `ringbuf_t` and ringbuf_local_t` structures.

* `int ringbuf_register(ringbuf_t *rbuf, ringbuf_local_t *rbuf_local)`
  * Register the current thread or process as a producer.  Each producer
  must register itself.  The `rbuf_local` is a pointer to an opauqe object
  storing information local to the producer, but accessible by the consumer.
  If using threads, this would be equivalent to the thread-local store (TLS);
  if using processes, this would typically be in shared memory.

* `ssize_t ringbuf_acquire(ringbuf_t *rbuf, ringbuf_local_t *rbuf_local, size_t len)`
  * Request a space of a given length in the ring buffer.  Returns the
  offset at which the space is available or -1 on failure.  Once the data
  is ready (typically, when writing to the ring buffer is complete), the
  `ringbuf_produce` function must be called to indicate that.

* `void ringbuf_produce(ringbuf_t *rbuf, ringbuf_local_t *rbuf_local)`
  * Indicate that the acquired range in the buffer is produced and is ready
  to be consumed.

* `size_t ringbuf_consume(ringbuf_t *rbuf, size_t *offset)`
  * Get a contiguous range which is ready to be consumed.  Returns zero
  if there is no data available for consumption.  Once the data is
  consumed (typically, when reading from the ring buffer is complete),
  the `ringbuf_release` function must be called to indicate that.

* `void ringbuf_release(ringbuf_t *rbuf, size_t nbytes)`
  * Indicate that the consumed range can now be released and may now be
  reused by the producers.

## Caveats

This ring buffer implementation always provides a contiguous range of
space for the producer.  It is achieved by early an wrap-around if the
requested range cannot fit in the end.  The implication of this is that
the `ringbuf_acquire` call may fail if the requested range is greater
than half of the buffer size.  Hence, it may be necessary to ensure that
the ring buffer size is at least twice as large as the maximum production
unit size.

## Example

Producers:
```c
if (ringbuf_register(r, tls) == -1)
	err(EXIT_FAILURE, "ringbuf_register")

...

if ((off = ringbuf_acquire(r, tls, len)) != -1) {
	memcpy(&buf[off], payload, len);
	ringbuf_produce(r, tls);
}
```

Consumer:
```c
if ((len = ringbuf_consume(r, &off)) != 0) {
	process(&buf[off], len);
	ringbuf_release(r, len);
}
```
