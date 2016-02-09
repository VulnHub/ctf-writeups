>> Get 10000 pt. This game is really hard, and so you can crack it.
>> http://adctf2014.katsudon.org/dat/JEZSnwooVJRYDRzt/shooting.html

The url in this challenge brought us to a javascript "space invader"-esque game. Looking at the source for http://adctf2014.katsudon.org/dat/JEZSnwooVJRYDRzt/shooting.min.js was kind of annoying since it was minified, so I used jsbeautifier to un-minify the code and I got the following:

```javascript
$UPcs4hr8oKgbbqAesfT = function(n) {
    if (typeof($UPcs4hr8oKgbbqAesfT.list[n]) == "string") return $UPcs4hr8oKgbbqAesfT.list[n].split("").reverse().join("");
    return $UPcs4hr8oKgbbqAesfT.list[n];
};
$UPcs4hr8oKgbbqAesfT.list = ["evomhcuot", "emarfretne", "gnp.cihparg", "kcalb", "tratshcuot", "dnehcuot"];
enchant();
window.onload = function() {
    gamé = new Game(320, 320);
    gamé.fps = 24;
    G = false;
    B = false;
    b = ["\x63", "\x68", "\x65", "\x65", "\x72", "\x75", "\x70", "\x2c", "\x20", "\x6b", "\x65", "\x65", "\x70", "\x20", "\x67", "\x6f", "\x69", "\x6e", "\x67", "\x21"];
    c = [107.4, 126.1, 131.2, 120.3, 130, 134.2, 129.1, 62.4, 55.5, 126.3, 133.3, 111.2, 120.2, 43.1, 122.3, 139.4, 123.5, 126, 123.6, 47.6, 19, 18.7, 18.8, 17.1, 20.6, 19.9, 17.9, 20.4, 17.5, 20.7, 20.2, 20.2];
    P = [b[0], b[1], b[2], b[3], b[4], b[5], b[6], b[7], b[8], b[9], b[10], b[11], b[12], b[13], b[14], b[15], b[16], b[17], b[18], b[19]];
    gamé.score = 0;
    gamé.touched = false;
    gamé.preload($UPcs4hr8oKgbbqAesfT(2));
    gamé.onload = function() {
        player = new f(0, 152);
        E = new Array;
        for (var e = 0; e < b.length; e++) {
            c[e] -= b[e].charCodeAt(0);
            c[e] = Math.round(c[e] * 10) / 10
        }
        gamé.rootScene.backgroundColor = $UPcs4hr8oKgbbqAesfT(3);
        gamé.rootScene.addEventListener($UPcs4hr8oKgbbqAesfT(1), function() {
            var e = 0;
            var t = gamé.score >= 5e3;
            if (gamé.score < 8e3 && (rand(1e3) < gamé.frame / 20 * Math.sin(gamé.frame / 100) + gamé.frame / 20 + 50 || t && rand(500) < gamé.frame / 20 * Math.sin(gamé.frame / 100) + gamé.frame / 20 + 50)) {
                var n = rand(320);
                var r = n < 160 ? .01 : -.01;
                (gamé.score >= 5e3) ? (e = Math.floor(Math.random() * 4)) : (e = Math.floor(gamé.score / 1e3));
                var i = new g(320, n, r, e);
                i.key = gamé.frame;
                E[gamé.frame] = i
            }
            if (B) {
                if (gamé.score >= 8e3) {
                    B = false;
                    for (var s in E) {
                        E[s].remove()
                    }
                    setTimeout(function() {
                        for (var e = 0; e < 20; e++) {
                            var t = rand(320);
                            var n = new h(300, e * 16, .01, 9999, c[e ^ b.length]);
                            n.key = e;
                            E[e] = n
                        }
                    }, 1e3)
                }
            }
            scoreLabel.score = gamé.score;
            (gamé.score == 1e4) ? ((!G) ? (G = true, setTimeout(function() {
                gamé.end();
                alert(P.join(""))
            }, 1e3)) : 0) : 0;
        });
        scoreLabel = new ScoreLabel(8, 8);
        gamé.rootScene.addChild(scoreLabel)
    };
    gamé.start()
};
var f = enchant.Class.create(enchant.Sprite, {
    initialize: function(e, t) {
        enchant.Sprite.call(this, 16, 16);
        this.image = gamé.assets[$UPcs4hr8oKgbbqAesfT(2)];
        this.x = e;
        this.y = t;
        this.frame = 0;
        gamé.rootScene.addEventListener($UPcs4hr8oKgbbqAesfT(4), function(e) {
            player.y = e.y;
            gamé.touched = true
        });
        gamé.rootScene.addEventListener($UPcs4hr8oKgbbqAesfT(0), function(e) {
            player.y = e.y
        });
        gamé.rootScene.addEventListener($UPcs4hr8oKgbbqAesfT(5), function(e) {
            player.y = e.y;
            gamé.touched = false
        });
        this.addEventListener($UPcs4hr8oKgbbqAesfT(1), function() {
            if (gamé.touched && gamé.frame % 3 == 0) {
                var e = new k(this.x, this.y)
            }
        });
        gamé.rootScene.addChild(this)
    }
});
var g = enchant.Class.create(enchant.Sprite, {
    initialize: function(e, t, n, r) {
        enchant.Sprite.call(this, 16, 16);
        this.image = gamé.assets[$UPcs4hr8oKgbbqAesfT(2)];
        this.x = e;
        this.y = t;
        this.level = r;
        switch (this.level) {
            case 0:
                this.frame = 3;
                break;
            case 1:
                this.frame = 4;
                break;
            case 2:
                this.frame = 5;
                break;
            default:
                this.frame = 6
        }
        this.omega = n;
        this.direction = 0;
        (this.level >= 3) ? (this.moveSpeed = 5) : (this.level >= 2) ? (this.moveSpeed = 4) : (this.moveSpeed = 3);
        this.addEventListener($UPcs4hr8oKgbbqAesfT(1), function() {
            this.move();
            if (this.y > 320 || this.x > 320 || this.x < -this.width || this.y < -this.height) {
                this.remove()
            } else if (this.level >= 3 && this.age % 5 == 0 || this.level < 3 && this.age % 10 == 0) {
                var e = new l(this.x, this.y)
            }
        });
        gamé.rootScene.addChild(this)
    },
    move: function() {
        this.direction += this.omega;
        this.x -= this.moveSpeed * Math.cos(this.direction / 180 * Math.PI);
        (this.level >= 1) ? ((player.y > this.y) ? (this.y += this.moveSpeed) : (player.y < this.y) ? (this.y -= this.moveSpeed) : 0) : 0;
    },
    remove: function() {
        gamé.rootScene.removeChild(this);
        delete E[this.key]
    }
});
var h = enchant.Class.create(g, {
    initialize: function(e, t, n, r, i) {
        g.call(this, e, t, n, r);
        this.image = gamé.assets[$UPcs4hr8oKgbbqAesfT(2)];
        this.x = e;
        this.y = t;
        this.c = i;
        this.level = r;
        this.life = 10;
        this.frame = 6;
        this.omega = n;
        this.direction = 0;
        this.clearEventListener($UPcs4hr8oKgbbqAesfT(1));
        this.addEventListener($UPcs4hr8oKgbbqAesfT(1), function() {
            if (this.age % 3 == 0) {
                var e = new l(this.x, this.y)
            }
        });
        gamé.rootScene.addChild(this)
    }
});
var j = enchant.Class.create(enchant.Sprite, {
    initialize: function(e, t, n) {
        enchant.Sprite.call(this, 16, 16);
        this.image = gamé.assets[$UPcs4hr8oKgbbqAesfT(2)];
        this.x = e;
        this.y = t;
        this.scaleX = -1;
        (n == 0) ? (this.frame = 1) : (this.frame = 2);
        this.direction = n;
        this.moveSpeed = 10;
        this.addEventListener($UPcs4hr8oKgbbqAesfT(1), function() {
            this.x += this.moveSpeed * Math.cos(this.direction);
            this.y += this.moveSpeed * Math.sin(this.direction);
            (this.y > 320 || this.x > 320 || this.x < -this.width || this.y < -this.height) ? (this.remove()) : 0;
        });
        gamé.rootScene.addChild(this)
    },
    remove: function() {
        gamé.rootScene.removeChild(this);
        delete this
    }
});
var k = enchant.Class.create(j, {
    initialize: function(e, t) {
        j.call(this, e, t, 0);
        this.addEventListener($UPcs4hr8oKgbbqAesfT(1), function() {
            for (var e in E) {
                (!B && E[e].intersect(this)) ? ((E[e].life !== undefined) ? (E[e].life--, this.remove(), (E[e].life == 0) ? (P[e] = String.fromCharCode(E[e].c * 10 ^ 255), gamé.score += 1000, E[e].remove()) : (E[e].life <= 3) ? (E[e].frame = 3) : (E[e].life <= 6) ? (E[e].frame = 4) : (E[e].life <= 8) ? (E[e].frame = 5) : 0) : (this.remove(), E[e].remove(), gamé.score += 1000, (gamé.score == 8e3) ? (B = true) : 0)) : 0;
            }
        })
    }
});
var l = enchant.Class.create(j, {
    initialize: function(e, t) {
        j.call(this, e, t, Math.PI);
        this.addEventListener($UPcs4hr8oKgbbqAesfT(1), function() {
            (player.within(this, 8)) ? (gamé.end()) : 0;
        })
    }
})

```

The hint to the challenge said "get 10000 points". So I looked around for anywhere that checked for 10000 points. I found it here:

```javascript

(gamé.score == 1e4) ? ((!G) ? (G = true, setTimeout(function() {
                gamé.end();
                alert(P.join(""))
            }, 1e3)) : 0) : 0;

```

Note: In Javascript, you can represent exponential numbers with the "e" modifier. Therefore, 1e4 is equivalent to 1 x 10^4.

There is an alert that is popped up once you reach 10,000 points. Unfortunately, this is a GOTCHYA and the alert box says "Cheer up, keep going!". The hint didn't really make sense since I reached 10,000 points and I was given a random message. However, I decided to look at other pieces of code that executed different actions depending on the number of points you received.

Then I read this piece of code, which looked interesting:

```javascript
var k = enchant.Class.create(j, {
    initialize: function(e, t) {
        j.call(this, e, t, 0);
        this.addEventListener($UPcs4hr8oKgbbqAesfT(1), function() {
            for (var e in E) {
                (!B && E[e].intersect(this)) ? ((E[e].life !== undefined) ? (E[e].life--, this.remove(), (E[e].life == 0) ? (P[e] = String.fromCharCode(E[e].c * 10 ^ 255), gamé.score += 1000, E[e].remove()) : (E[e].life <= 3) ? (E[e].frame = 3) : (E[e].life <= 6) ? (E[e].frame = 4) : (E[e].life <= 8) ? (E[e].frame = 5) : 0) : (this.remove(), E[e].remove(), gamé.score += 1000, (gamé.score == 8e3) ? (B = true) : 0)) : 0;
            }
        })
    }
});
```
(P[e] = String.fromCharCode(E[e].c * 10 ^ 255) seems to be "decoding" some potentially secret data when the score is 8,000 points. It turns out, I was correct. So playing the game until 8,000 points, I used the following javascript in my browser console to dump the contents and get the flag:

```javascript
var word = '';
for(var e in E){
word += String.fromCharCode(E[e].c * 10 ^ 255);
}

console.log(word);
```

Flag: ADCTF_1mP05518L3_STG
