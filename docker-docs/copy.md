## COPY

This command copies entire application code from local machine into container's working directory.

```sh
COPY source__data_to_be_copied destination_of_copied_file_to_be_saved.
```

```sh
COPY . .
```

We have to make sure that the `Dockerfile` is located in our root directory.
What we are doing using above command is copying entire application code from root directory into current working directory(`app`).
