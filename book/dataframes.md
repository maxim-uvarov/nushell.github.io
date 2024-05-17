# Dataframes

::: warning
Starting with version 0.93, there is a more recent implementation of dataframes in `nu_plugin_polars`, which includes a newer version of `polars` and many bug fixes.
From version 0.94 of Nushell, internal dataframes (with the `polars` prefix) are going to be deprecated in favor of `nu_plugin_polars`.

To use `nu_plugin_polars`, you'll need to install [cargo](https://doc.rust-lang.org/cargo/getting-started/installation.html) and then install the plugin with commands:

```nu no-run
# Install the `polars` nushell plugin
> cargo install nu_plugin_polars

# Add the plugin's commands to your plugin registry file:
> plugin add ~/.cargo/bin/nu_plugin_polars
```

After installation, you will need to restart the nushell instance. If everything is successful,
you should be able to see command completions for `polars`. For example, you should be able to execute
`polars into-df -h`.

:::

As we have seen so far, Nushell makes working with data its main priority.
`Lists` and `Tables` are there to help you cycle through values in order to
perform multiple operations or find data in a breeze. However, there are
certain operations where a row-based data layout is not the most efficient way
to process data, especially when working with extremely large files. Operations
like group-by or join using large datasets can be costly memory-wise, and may
lead to large computation times if they are not done using the appropriate
data format.

For this reason, the `DataFrame` structure was introduced to Nushell. A
`DataFrame` stores its data in a columnar format using as its base the [Apache
Arrow](https://arrow.apache.org/) specification, and uses
[Polars](https://github.com/pola-rs/polars) as the motor for performing
extremely [fast columnar operations](https://h2oai.github.io/db-benchmark/).

You may be wondering now how fast this combo could be, and how could it make
working with data easier and more reliable. For this reason, let's start this
page by presenting benchmarks on common operations that are done when
processing data.

## Benchmark comparisons

For this little benchmark exercise we will be comparing native Nushell
commands, dataframe Nushell commands and [Python
Pandas](https://pandas.pydata.org/) commands. For the time being don't pay too
much attention to the [`Dataframe` commands](/commands/categories/dataframe.md). They will be explained in later
sections of this page.

> System Details: The benchmarks presented in this section were run using a
> machine with a processor Intel(R) Core(TM) i7-10710U (CPU @1.10GHz 1.61 GHz)
> and 16 gb of RAM.
>
> All examples were run on Nushell version 0.33.1.
> (Command names are updated to Nushell 0.78)

### File information

The file that we will be using for the benchmarks is the
[New Zealand business demography](https://www.stats.govt.nz/assets/Uploads/New-Zealand-business-demography-statistics/New-Zealand-business-demography-statistics-At-February-2020/Download-data/Geographic-units-by-industry-and-statistical-area-2000-2020-descending-order-CSV.zip) dataset.
Feel free to download it if you want to follow these tests.

The dataset has 5 columns and 5,429,252 rows. We can check that by using the
`polars store-ls` command:

```nu
> let df = polars open Data7602DescendingYearOrder.csv
> polars store-ls
╭──────────────────────────────────────┬─────────┬─────────┬─────────┬───────────┬────────────────┬───────────────┬────────────┬──────────┬─────────────────╮
│                 key                  │ created │ columns │  rows   │   type    │ estimated_size │ span_contents │ span_start │ span_end │ reference_count │
├──────────────────────────────────────┼─────────┼─────────┼─────────┼───────────┼────────────────┼───────────────┼────────────┼──────────┼─────────────────┤
│ 1b26610a-1319-400e-b724-2f3baa0fc6c5 │ now     │       5 │ 5429252 │ DataFrame │       184.5 MB │ polars open   │    1986861 │  1986872 │               1 │
╰──────────────────────────────────────┴─────────┴─────────┴─────────┴───────────┴────────────────┴───────────────┴────────────┴──────────┴─────────────────╯
```

We can have a look at the first lines of the file using [`first`](/commands/docs/first.md):

```nu
> $df | polars first
╭───┬──────────┬─────────┬──────┬───────────┬──────────╮
│ # │ anzsic06 │  Area   │ year │ geo_count │ ec_count │
├───┼──────────┼─────────┼──────┼───────────┼──────────┤
│ 0 │ A        │ A100100 │ 2000 │        96 │      130 │
╰───┴──────────┴─────────┴──────┴───────────┴──────────╯
```

...and finally, we can get an idea of the inferred data types:

```nu
> $df | polars schema
╭───────────┬─────╮
│ anzsic06  │ str │
│ Area      │ str │
│ year      │ i64 │
│ geo_count │ i64 │
│ ec_count  │ i64 │
╰───────────┴─────╯
```

### Loading the file

Let's start by comparing loading times between the various methods. First, we
will load the data using Nushell's [`open`](/commands/docs/open.md) command:

```nu
> timeit {open Data7602DescendingYearOrder.csv}
1sec 648ms 940µs 959ns
```

Loading the file using native Nushell functionality took 1.63 seconds. Not bad for
loading five million records! But we can do a bit better than that.

Let's now use Pandas. We are going to use the next script to load the file:

```nu
('import pandas as pd

df = pd.read_csv("Data7602DescendingYearOrder.csv")'
| save load.py -f)
```

And the benchmark for it is:

```nu
> timeit {python load.py}
1sec 331ms 548µs 750ns
```

Here bare nushell goes almost like pandas!

Probably we can load the data a bit faster. This time we will use Nushell's
`polars open` command:

```nu
> timeit {polars open Data7602DescendingYearOrder.csv; null}
9sec 157ms 826µs 666ns
```
