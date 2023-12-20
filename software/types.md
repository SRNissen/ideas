wrapper for fundamental types that define overflow behavior

Same as ^prev, with custom upper/lower bound

Same as 2x ^prev^ but with arbitrary types and arbitrary constraints

wrapper for heap object that's like unique_ptr but copyable (copies the underlying object, essentially like a std::vector that only has space for 1 object)

wrapper for mutable object that does the thread safety for you so you can e.g. have a const method update the a cache object without data races


