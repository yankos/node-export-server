# Highcharts Node.js Export Server

The export server can be ran as both a CLI converter, and as a HTTP server.

## Install
    
    npm install highcharts-export-server -g

OR:
    
    git clone https://github.com/highcharts/node-export-server
    npm install
    npm link


## Running
    
    highcharts-export-server <arguments>

## Command Line Arguments
    
  * `--infile`: Specify the input file.
  * `--instr`: Specify the input as a string.
  * `--options`: Alias for `--instr`
  * `--outfile`: Specify the output filename.
  * `--type`: The type of the exported file. Valid options are `jpg png pdf svg`.
  * `--scale`: The scale of the chart.
  * `--width`: Scale the chart to fit the width supplied - overrides `--scale`.
  * `--constr`: The constructor to use. Either `Chart` or `StockChart`.
  * `--callback`: File containing JavaScript to call in the constructor of Highcharts.
  * `--resources`: Stringified JSON.
  * `--host`: The hostname to run a server on.
  * `--port`: The port to listen for incoming requests on.
  * `--tmpdir`: The path to temporary output files.
  * `--sslPath`: The path to the SSL key/certificate. Indirectly enables SSL support.
  * `--enableServer <1|0>`: Enable the server (done also when supplying --host)
  * `--logDest <path>`: Set path for log files, and enable file logging
  * `--logLevel <0..4>`: Set the log level. 0 = off, 1 = errors, 2 = warn, 3 = notice, 4 = verbose
  * `--batch "input.json=output.png;input2.json=output2.png;.."`: Batch convert

`-` and `--` can be used interchangeably when using the CLI.

## Setup: Injecting the Highcharts dependency

In order to use the export server, Highcharts.js needs to be injected
into the export template.

This is largely an automatic process. When running `npm install` you will
be prompted to accept the license terms of Highcharts.js. Answering `yes` will
pull the latest source from the Highcharts CDN and put them where they need to be.

However, if you need to do this manually you can run `node build.js`.

## HTTP Server

The server accepts the following arguments:

  * `infile`: A string containing JSON or SVG for the chart 
  * `options`: Aliast for `infile`
  * `svg`: A string containing SVG to render
  * `type`: The format: `png`, `jpeg`, `pdf`, `svg`. Mimetypes can also be used.
  * `scale`: The scale factor
  * `width`: The chart width (overrides scale)
  * `callback`: Javascript to execute in the highcharts constructor.
  * `resources`: Additional resources.
  * `constr`: The constructor to use. Either `Chart` or `Stock`.
  * `b64`: Bool, set to true to get base64 back instead of binary.
  * `async`: Get a download link instead of the file data
  * `noDownload`: Bool, set to true to not send attachment headers on the response.

Note that the `b64` option overrides the `async` option.

It responds to `application/json`, `multipart/form-data`, and URL encoded requests.

CORS is enabled for the server.

It's recommended to run the server using [forever](https://github.com/foreverjs/forever).

### SSL

To enable ssl support, drop your `server.key` and `server.crt` in the ssl folder,
or add `--sslPath <path to key/crt>` when running the server.

## Server Test

Run the below in a terminal after running `highcharts-export-server --enableServer`.
    
    # Generate a chart and save it to mychart.png    
    curl -H "Content-Type: application/json" -X POST -d '{"infile":{"title": {"text": "Steep Chart"}, "xAxis": {"categories": ["Jan", "Feb", "Mar"]}, "series": [{"data": [29.9, 71.5, 106.4]}]}}' 127.0.0.1:7801 -o mychart.png

## Using as a Node.js Module

The export server can also be used as a node module to simplify integrations:
    
    //Include the exporter module
    const exporter = require('highcharts-export-server');

    //Export settings 
    var exportSettings = {
        type: 'png',
        options: {
            title: {
                text: 'My Chart'
            },
            xAxis: {
                categories: ["Jan", "Feb", "Mar", "Apr", "Mar", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"]
            },
            series: [
                {
                    type: 'line',
                    data: [1, 3, 2, 4]
                },
                {
                    type: 'line',
                    data: [5, 3, 4, 2]
                }
            ]
        }
    };

    //Set up a pool of PhantomJS workers
    exporter.initPool();

    //Perform an export
    /*
        Export settings corresponds to the available CLI arguments described
        above.
    */
    exporter.export(exportSettings, function (err, res) {
        ...
        //Kill the pool when we're done with it
        exporter.killPool();
    });

### Node.js API Reference

**highcharts-export-server module**

**Functions**

  * `log(level, ...)`: log something. Level is a number from 1-4. Args are joined by whitespace to form the message.
  * `logLevel(level)`: set the current log level: `0`: disabled, `1`: errors, `2`: warnings, `3`: notices, `4`: verbose
  * `enableFileLogging(path, name)`: enable logging to file. `path` is the path to log to, `name` is the filename to log to
  * `export(exportOptions, fn)`: do an export. `exportOptions` uses the same attribute names as the CLI switch names. `fn` is called when the export is completed, with an object as the second argument containing the the filename attribute.
  * `startServer(port, sslPort, sslPath)`: start an http server on the given port. `sslPath` is the path to the server key/certificate (must be named server.key/server.crt)
  * `initPool(config)`: init the phantom pool - must be done prior to exporting. `config` is an object as such:
    * `maxWorkers` (default 25) - max count of worker processes
    * `initialWorkers` (default 5) - initial worker process count
    * `workLimit` (default 50) - how many task can be performed by a worker process before it's automatically restarted
  * `killPool()`: kill the phantom processes

## Performance Notice

In cases of batch exports, it's faster to use the HTTP server than the CLI.
This is due to the overhead of starting PhantomJS for each job when using the CLI. 

As a concrete example, running the CLI with [testcharts/basic.json](testcharts/basic.json) 
as the input and converting to PNG averages about 449ms. 
Posting the same configuration to the HTTP server averages less than 100ms.

So it's better to write a bash script that starts the server and then
performs a set of POSTS to it through e.g. curl if not wanting to host the
export server as a service.

Alternatively, you can use the `--batch` switch if the output format is the same
for each of the input files to process:
    
    highcharts-export-server --batch "infile1.json=outfile1.png;infile2.json=outfile2.png;.."

Other switches can be combined with this switch.

## License

[MIT](LICENSE).