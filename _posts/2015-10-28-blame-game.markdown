---
layout: post
title:  "Blame game with libgit2"
date:   2015-10-28 20:11:28
categories: git ruby quality
---
If you've been working in the industry long enough you've probably long established that providing objective performance criteria for programmers is impossible and - if attempted - harmful. We all have our heuristics though - sometimes there is a piece of code that you discover that makes you doubt the human reason (BTW how often do you discover it's actually yours?). In one of the places I worked there was a special internal group called _Clowntown_ that was exclusively dedicated to sharing these little gems. So yes, we can established that sometimes perfectly reasonable people can agree on the quality of the code. Now if you want to make sure whether the laugh is not you you can always run `git blame` on the file.

Parking `git blame` for a while - since we will return to it later - let's focus on refactoring. Depending on which side you're on it's either a complete waste of time (business) or a necessary measure to keep the code maintainable (programmers). The usual question business asks is - if the code is so broken and someone wrote it (programmers) why would the same people be qualified enough to fix it. As coders we all know that this is a wrong question to ask but... coming to think of it... is it, really?

The point here is - if you keep rewriting the same piece of code over and over it probably is a testament to the ability of the person writing the original code in the first place - whether it is yourself of someone else. We can easily figure out how many lines of code any given coder committed at any given point in time (commit) using the output of `git blame`. If we compare these numbers for two adjacent commits we know whose code was deleted (and by whom). We can then aggregate those numbers to get output like this:

{% highlight bash %}
+---------+-------------+------------+---------------+
|         | Alice [148] | Bob [1447] | Charlie [736] |
+---------+-------------+------------+---------------+
| Alice   | 0.00%       | 0.76%      | 7.07%         |
| Bob     | 0.68%       | 8.09%      | 29.35%        |
| Charlie | 59.46%      | 4.15%      | 25.00%        |
+---------+-------------+------------+---------------+
{% endhighlight %}

Some explanation - each row shows who deleted code, and each column shows whose
code was deleted. For example Alice did not delete any of her code, but she deleted 0.76% of Bob's code (1447 lines written in total) and 7.07% of Charlie's code (736 lines written in total).

The script that generated this output programmatically accesses Git via Ruby's `libgit2` bindings using a gem called `rugged`. I hacked it in one evening without knowing first thing about `libgit2` so bear with me:

{% highlight ruby %}
require 'find'
require 'pathname'
require 'rugged'
require 'terminal-table'

def main
  repo_path = Pathname.new(ARGV[0] || '.')
  extname = ARGV[1] || '.rb'
  branch = ARGV[2] || 'master'
  repo = Rugged::Repository.new(repo_path.to_s)
  walker = Rugged::Walker.new(repo).tap do |w|
    w.sorting(Rugged::SORT_DATE)
    w.push(repo.references["refs/heads/#{branch}"].target)
  end
  trace = walker.each_with_object([]) do |commit, collection|
    repo.checkout(commit.oid, strategy: :force)
    stats = Hash.new(0)
    Find.find(ARGV[0]).each do |p|
      Find.prune if File.basename(p).start_with?('.')
      next if Pathname(p).extname != extname
      relative = Pathname.new(p).relative_path_from(repo_path)
      blame = Rugged::Blame.new(repo, relative.to_s)
      blame.each do |hunk|
        author_name = hunk[:final_signature][:name].split(' ').first
        stats[author_name] += hunk[:lines_in_hunk]
      end
    end
    commiter_name = commit.author[:name].split(' ').first
    collection << [commiter_name, stats, commit.oid]
    print '.' # because it takes aaaaaages...
  end
  print "\n\n\n"
  trace.reverse!
  authored = Hash.new(0)
  removals = Hash.new { |hash, key| hash[key] = {} }
  previous_stats = trace[0][1]
  trace[1..-1].each do |(author, stats, _)|
    diff = {}.update(stats).update(previous_stats) { |_, o, n| o - n }
    authored[author] += diff[author] if (diff[author] || 0) > 0
    removals[author]
      .update(diff.select { |k, v| k != author || v < 0 }) { |_, o, n| o + n }
    previous_stats = stats
  end
  vs_total = removals.to_a.map do |author, lines_removed|
    proportion = lines_removed.to_a.map do |target, lines|
      [target, lines.abs.to_f / authored[target] * 100]
    end.to_h
    [author, proportion]
  end.to_h
  rows = []
  people = vs_total.keys.sort
  people.each do |person|
    row = [person]
    people.each_with_object(row) do |target, r|
      percent = vs_total[person][target] || 0.0
      r << "#{format('%.2f', percent)}%"
    end
    rows << row
  end
  headings = [''] + people.map { |person| "#{person} [#{authored[person]}]" }
  puts Terminal::Table.new(headings: headings, rows: rows)
end

main if __FILE__ == $PROGRAM_NAME
{% endhighlight %}

There is a ton of bad things about this code, starting with a massive undocumented `main` function I would reject in code review and hash-based programming I generally despise. But it vörks... kind of. See, because it actually checks out every commit and then runs its checks on individual files it does things to your filesystem and you're heavily blocked on your IO. To give you an idea of how slow the script actually is - analysing a small repo with just 64 commits in a Docker container given one core and 4GB of RAM took 18 seconds. Which is aaaaaages in software time because - you know - computers are fast these days. I realised I was blocked on IO so I went to the store to see if they had a faster disk. They didn't. And I didn't really go to the store. I used a ramdisk instead:

{% highlight bash %}
$ mkdir /mnt/ramdisk
$ mount -t tmpfs -o size=1G tmpfs /mnt/ramdisk
{% endhighlight %}

That alone cut down the time from 18 seconds to 1.6 seconds which isn't too bad but isn't great either. When running on a huge project it took 160 minutes (I know, right?) and then it crashed with a Ruby double free bug at the end. So yes, it kind of sucks. There is surely a way to make it snappy because if you think about it the algorithm isn't that complex - `O(n^2 * m * p)` at worst where `n` is the number of commits, `m` is the number of files to analyse and `p` is the number of contributors. So if I knew Git internals the way I don't know them I'm fairly sure I could get reasonable analysis times. One day perhaps I'll even turn it into a product.

Techicalities aside you can easily imagine how such an input (in conjunction with other metrics) could be used to determine performance problems on a software team. If most of a given person's code is being deleted by others it may provide an indicator that something is amiss. Perhaps with more training and guidance they can find their way out of this conundrum?

Being a smart person that you are you know that statistics don't tell the whole story. Someone way smarter than myself once said that _individuals use statistics as a drunk man uses lamp-posts - for support rather than for illumination_. So I suppose what I'm trying to say is - don't be that person.



