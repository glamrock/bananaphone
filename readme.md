<h1>Bananaphone: Reverse Hash Encoding</h1>

Leif Ryge <leif@synthesize.us>
No Rights Reserved, Uncopyright 2011

This is an implementation of a steganographic encoding scheme which transforms
a stream of binary data into a stream of tokens (eg, something resembling
natural language text) such that the stream can be decoded by concatenating the
hashes of the tokens.

<b>TLDR</b>: SSH over Markov chains

The encoder is given a word size (number of bits), a tokenization function (eg,
split text on whitespace), a hash function (eg, sha1), a corpus, and a modeling
function (eg, a markov model, or a weighted random model). The range of the
hash function is truncated to the word size. The model is built by tokenizing
the corpus and hashing each token with the truncated hash function. For the
model to be usable, there must be enough tokens to cover the entire hash space
(2^(word size) hashes). After the model is built, the input data bytes are
scaled up or down to the word size (eg, scaling [255, 18] from 8-bit bytes to
4-bit words produces [15, 15, 1, 2]) and finally each scaled input word is
encoded by asking the model for a token which hashes to that word. (The
encoder's model can be thought of as a probabilistic reverse hash function.)


<h2>FUTURE IDEAS</h2>
This was written as a fun experimental proof-of-concept hack, and in its
present form it is TRIVIALLY DETECTABLE since there is no encryption used.
(When you ssh over it, an observer only needs to guess your tokenization
function, hash function and word size to see that you are using ssh.) It would
be easy enough to add some shared-secret crypto, but the beginning of an SSH or
SSL session (and certain activities within them) would still be detectable
using timing analysis.

I have a vague idea for another layer providing timing cover for binary streams
(adding traffic to make the apparent throughput constant or randomly varied);
To make a steganographic stream encoder which is actually resistant to
steganalysis, perhaps encryption should be applied on top of that layer (under
this one? Note: these are tricky problems and I don't really know what I'm
doing :)

Or instead, maybe this could be layered under the Tor Project's obfsproxy?

Another idea: For encoding short messages, one could use the markov model and
the weighted random model to provide an auto-completing text-entry user
interface where entirely innocuous (human-meaningful) cover messages could be
composed.


<h2>USAGE NOTES</h2>
The tokenization function needs to produce tokens which will always re-tokenize
the same way after being concatenated with each other in any order. So, for
instance, a "split on whitespace" tokenizer actually needs to append a
whitespace character to each token. The included "words" tokenizer replaces
newlines with spaces; "words2" does not, and "words3" does sometimes. The other
included tokenizers, "lines", "bytes", and "asciiPrintableBytes" should be
self-explanatory.

For streaming operation, the word size needs to be a factor or multiple of 8.
(Otherwise, bytes will frequently not be deliverable until after the subsequent
byte has been sent, which breaks most streaming applications). Implementing the
above-mentioned layer of timing cover would obviate this limitation. Also, when
the word size is not a multiple or factor of 8, there will sometimes be 1 or 2
null bytes added to the end of the message (due to ambiguity when converting
the last word back to 8 bits).

The markov encoder supports two optional arguments: the order of the model
(number of previous tokens which constitute a previous state, default is 1),
and --abridged which will remove all states from the model which do not lead to
complete hash spaces. If --abridged is not used, the markov encoder will
sometimes have no matching next token and will need to fall back to using the
random model. If -v is specified prior to the command, the rate of model
adherence is written to stderr periodically. With a 3MB corpus of about a half
million words (~50000 unique), at 2 bits per word (as per the SSH example
below) the unabridged model is adhered to about 90% of the time.


<h2>EXAMPLE USAGE:</h2>

encode <i>"Hello\n"</i> at 13 bits per word, using a dictionary and random picker:
<code>    echo Hello | ./bananaphone.py rh_encoder words,sha1,13 random /usr/share/dict/words</code>

decode <i>"Hello\n"</i> from 13-bit words:
<code>    echo "discombobulate aspens brawler Gödel's" | ./bananaphone.py rh_decoder words,sha1,13</code>

start a proxy listener for <i>$sshhost</i>, using markov encoder with 2 bits per word:
<code>    socat TCP4-LISTEN:1234,fork EXEC:'bash -c "./bananaphone.py\\ rh_decoder\\ words,sha1,2|socat\\ TCP4\:'$sshhost'\:22\\ -|./bananaphone.py\\ -v\\ rh_encoder\\ words,sha1,2\\ markov\\ corpus.txt"'</code>

connect to the ssh host through the <i>$proxyhost</i>:
<code>    ssh user@host -oProxyCommand="./bananaphone.py rh_encoder words,sha1,2 markov corpus.txt|socat TCP4:$proxyhost:1234 -|./bananaphone.py rh_decoder words,sha1,2"</code>

