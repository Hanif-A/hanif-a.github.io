
```python

def get_duration(start_time, end_time, return_format=None):

    # start_time: datetime object
    # end_time: datetime object
    # See below for acceptable return_formats

    # Duration in seconds
    duration = int((end_time-start_time))

    # If a return_format was provided
    if return_format:
        if return_format == 's':
            
            # Seconds, do nothing, as it's already in seconds
            duration = f'{int(duration)} seconds'

        elif return_format == 'm':

            # Whole minutes
            duration = f'{int(duration / 60)} minutes'

        elif return_format == 'ms':

            # Minutes and seconds
            mins = int(duration / 60)
            duration = f'{mins}m {duration-(mins*60)}s'

    
    return duration
    
    ```
