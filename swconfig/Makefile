#  SPDX-License-Identifier: GPL-2.0

CFLAGS = -O2 -g -Iinclude -I/usr/include/libnl3 -D_GNU_SOURCE
LIBS=-lnl-3 -lnl-genl-3

all: swconfig

%.o: %.c
	$(CC) $(CFLAGS) -fPIC -c -o $@ $^

swconfig: swlib.o cli.o # uci.o
	$(CC) $(LDFLAGS) -o $@ $^ $(LIBS)
