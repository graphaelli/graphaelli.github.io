+++
title = "Guess the utility"
date = 2018-03-19T22:22:22-05:00
+++

{{< rawhtml >}}

<style>
.description { cursor: pointer; }
.solution { cursor: pointer; }
</style>
<script>
desc1 = "run a program with a time limit";
solution1 = "<code>/usr/bin/timeout</code>. The real timeout will send term first, and only optionally send kill after -k duration.  This sends kill from the start.";

desc2 = "run a program and summarize system resource usage";
solution2 = "<code>/usr/bin/time --format='rss:%M user:%U sys:%S real:%E'</code>.  Not the bash builtin, <a href='http://git.savannah.gnu.org/cgit/time.git/tree/src/time.c'>time.c</a>.  Similar to /usr/bin/time -l on mac os.";

desc3 = "run a program with process address space resource limit";
solution3 = "<code>/usr/bin/prlimit -v</code>";
</script>

{{< /rawhtml >}}

I invented a couple of utilities at work today. They're really useful little go programs. So useful, in fact,
that they have already existed for many years. I haven't written in a while so I figured I'd post them here as a little
guessing game. See if you can figure out what they are from the code, just note that they are not faithful
reimplementations, as I didn't originally set out to reimplement anything. Expand the description for a hint.

{{< rawhtml >}}

<h3>#1</h3>
{{< /rawhtml >}}

{{< highlight go >}}
{
        ctx, _ := context.WithTimeout(context.Background(), timeout)
        cmd := exec.CommandContext(ctx, os.Args[1], os.Args[2:]...)
        cmd.Stdout, cmd.Stderr, cmd.Stdin = os.Stdout, os.Stderr, os.Stdin
        var err error
        if err = cmd.Start(); err != nil {
                fmt.Fprintln(os.Stderr, err)
        } else {
                err = cmd.Wait()
        }
        os.Exit(exitCode(err))
}
{{< /highlight >}}

{{< rawhtml >}}

<p class="description" onClick="this.innerHTML=desc1">description</p>
<p class="solution" onClick="this.innerHTML=solution1">solution</p>

<h3>#2</h3>
{{< /rawhtml >}}

{{< highlight go >}}
{
	cmd := exec.Command(os.Args[1], os.Args[2:]...)
	cmd.Stdout, cmd.Stderr, cmd.Stdin = os.Stdout, os.Stderr, os.Stdin
	var err error
	start := time.Now()
	if err = cmd.Start(); err != nil {
		fmt.Fprintln(os.Stderr, err)
	} else {
		err = cmd.Wait()
	}
	end := time.Now()
	var rusage syscall.Rusage
	syscall.Getrusage(syscall.RUSAGE_CHILDREN, &rusage)
	fmt.Printf("rss:%d user:%s sys:%s real:%s\n", rusage.Maxrss, time.Duration(rusage.Utime.Nano())*time.Nanosecond, time.Duration(rusage.Stime.Nano())*time.Nanosecond, end.Sub(start))
	os.Exit(exitCode(err))
}
{{< /highlight >}}

{{< rawhtml >}}

<p class="description" onClick="this.innerHTML=desc2">description</p>
<p class="solution" onClick="this.innerHTML=solution2">solution</p>

<h3>#3</h3>
{{< /rawhtml >}}

{{< highlight go >}}
{
       var rlimit syscall.Rlimit
       rlimit.Cur = 134217728
       rlimit.Max = 134217728
       syscall.Setrlimit(syscall.RLIMIT_AS, &rlimit)
       cmd := exec.Command(os.Args[1], os.Args[2:]...)
       cmd.Stdout, cmd.Stderr, cmd.Stdin = os.Stdout, os.Stderr, os.Stdin
       var err error
       if err = cmd.Start(); err != nil {
               fmt.Fprintln(os.Stderr, err)
       } else {
               err = cmd.Wait()
       }
       os.Exit(exitCode(err))
}
{{< /highlight >}}

{{< rawhtml >}}

<p class="description" onClick="this.innerHTML=desc3">description</p>
<p class="solution" onClick="this.innerHTML=solution3">solution</p>
{{< /rawhtml >}}

If you like this kind of thing, many other utilities can be found in [go-coreutils](https://github.com/ericlagergren/go-coreutils).
