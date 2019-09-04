top_x_by = (tables=<-, by_measurement, x) =>
  tables
  |> group(columns:[by_measurement,"_measurement"], mode:"by")
  |>sort(columns:["_value"],desc:true)
  |>limit(n:1)
  |>group()
  |>sort(columns:["_value"],desc:true)
  |>limit(n:x)
  |>rename(columns: {_time: "ignore_time"})


hosts_ranked_by_cpu = from(bucket: "telegraf/autogen")
  |> range(start: dashboardTime)
  |> filter(fn: (r) => 
  	r._measurement == "cpu" and 
    r._field == "usage_user"
  )
  |>max()
  |>top_x_by(by_measurement:"host",x:2)

all_hosts_cpu_series = from(bucket: "telegraf/autogen")
  |> range(start: dashboardTime)
  |> filter(fn: (r) => 
  	r._measurement == "cpu" and 
    r._field == "usage_user"
  )
  |>rename(columns: {_start: "start", _stop: "stop"})
  |>group()
  
join(
  tables: {cpu:hosts_ranked_by_cpu, all:all_hosts_cpu_series},
  on: ["host"]
)
|>group(columns:["host"], mode: "by")
|>map(fn:(r) => ({
		_time: r._time,
        _value: r._value_all,
    }))
