#Original tweet from VictoriqueM

Just finished the Billion Row Challenge in Go. Had to process a billion temperature readings and honestly the whole thing was a pain in the arse but whatever

So the problem is you've got this 14GB file with a billion lines and you need to calculate min/max/average temps for each weather station. Most people probably just read the file normally like idiots so I figured I'd do something less stupid.

Fuck reading files the normal way. I wrote custom memory mapping using raw syscalls because copying 14GB into RAM is stupid and if you do it, you should feel bad. Just map the entire cunt directly into virtual memory.

Had to write separate implementations for Unix and Windows because apparently I hate myself. The Windows CreateFileMapping API is particularly annoying but whatever, it works (and seems to actually be faster than Unix).

The tricky bit was splitting this massive file across CPU cores without cutting lines in half like a numbnuts. Had to write logic that finds newline boundaries so each worker gets a clean chunk to process. Boring but necessary.

Threw out the entire Go string parsing library and wrote my own temperature parser that makes a bunch of assumptions and probably breaks on edge cases. But it's fast as shit and handles the expected format fine so who cares.

The parallelization is pretty straightforward - spawn one goroutine per CPU core, let them tear through their chunks, then merge all the results at the end. No fancy synchronization needed, just raw parallel processing.

Each worker maintains its own map of station stats then I combine everything once they're done. The whole thing processes like 400 million rows per second and reads data at 5+ GB/s which is apparently fast.

Zero external dependencies because I am either too much into code or i am too egotistic and the Go stdlib is good enough for this shit. Just syscalls and some creative abuse of unsafe pointers.

Also built a data generator that creates realistic test files with actual weather station names. That was almost more annoying than the actual challenge because generating a billion rows of realistic data takes forever.

The cross-platform stuff was tedious was a bit shit - different memory mapping APIs, different file handling, different everything. But now it runs on whatever so I guess that's something, does this mean it's not ENTIRELY cross compatible? i tested it with linux, windows and macos ARM and it all works fine

Anyway it works and it's fast and I'm done thinking about parsing temperature strings for a while. 

15 seconds to generate and about 4.8 to parse, considering that the fastest Java version i could find that didn't fuck around with some low level assembly was 10 seconds, i think it's not bad for 2 hours or so of coding.

# Billion Row Challenge - Go Implementation

A high-performance Go implementation of the [Billion Row Challenge](https://github.com/gunnarmorling/1brc) that processes one billion temperature measurements as fast as possible.

## Overview

This implementation processes a text file containing one billion rows of weather station temperature data in the format:
```
Station Name;Temperature
Hamburg;12.0
Bulawayo;8.9
Palembang;38.8
```

## Features

- **Memory-mapped file I/O** for maximum performance
- **Multi-threaded processing** using all available CPU cores
- **Cross-platform support** (Unix/Linux and Windows)
- **Custom temperature parser** optimized for the expected format
- **Data generator** to create test files with realistic weather station data
- **Zero external dependencies** - uses only Go standard library

## Usage

### Generate Test Data

First, you'll need a `weather_stations.csv` file containing weather station names (one per line or semicolon-separated). Then generate the billion-row dataset:

```bash
go run . -generate
```

This creates a `data.txt` file with 1 billion temperature measurements (~13-14 GB).

### Process the Data

Run the challenge processor:

```bash
go run .
```

The program will output results in the format:
```
{Abha=-23.0/18.0/59.2, Abidjan=-16.2/26.0/67.3, Abéché=-10.0/29.4/69.0, ...}

RESULTS
Total Time: 5.1364809s
Speed: 194.69 million rows/second
I/O Rate: 3.05 GB/second
```

## Performance Optimizations

- **Memory mapping**: Direct file access without copying data into memory
- **Parallel processing**: Work is distributed across all CPU cores
- **Custom parsing**: Hand-optimised temperature string parsing
- **data structures**: Pre-allocated maps and minimal allocations
- **based processing**: File is split into worker-sized chunks at line boundaries

## Architecture

### Core Components

- `main.go` - Main processing logic and coordination
- `generator.go` - Test data generation
- `mmap_unix.go` / `mmap_windows.go` - Platform-specific memory mapping

### Processing Flow

1. Memory-map the input file for zero-copy access
2. Split file into chunks aligned on line boundaries
3. Process chunks in parallel across CPU cores
4. Parse station names and temperatures from each line
5. Accumulate statistics (min/max/sum/count) per station
6. Merge results from all workers
7. Sort stations alphabetically and output results

## Building

```bash
# Build executable
go build -o billion-rows .

# Run with generated binary
./billion-rows -generate  # Generate data
./billion-rows            # Process data
```

## Performance Notes

Typical performance on modern hardware:
- **Generation**: 50-100 million rows/second
- **Processing**: 300-500 million rows/second
- **I/O throughput**: 4-8 GB/second

Performance scales with:
- Number of CPU cores
- Memory bandwidth
- Storage I/O speed
