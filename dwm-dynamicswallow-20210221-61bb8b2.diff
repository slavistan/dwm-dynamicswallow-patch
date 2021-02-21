From ac7558081ea5f576ef2ed09c1817d2722baa92fd Mon Sep 17 00:00:00 2001
From: =?UTF-8?q?Stanislaw=20H=C3=BCll?= <st@nislaw.de>
Date: Sun, 21 Feb 2021 10:27:10 +0100
Subject: [PATCH] dynamicswallow patch

---
 Makefile     |   3 +
 config.def.h |   7 +
 dwm.c        | 602 +++++++++++++++++++++++++++++++++++++++++++++++++--
 dwmswallow   | 120 ++++++++++
 util.c       |  30 +++
 util.h       |   1 +
 6 files changed, 740 insertions(+), 23 deletions(-)
 create mode 100755 dwmswallow

diff --git a/Makefile b/Makefile
index 77bcbc0..8bd79c8 100644
--- a/Makefile
+++ b/Makefile
@@ -40,12 +40,15 @@ install: all
 	mkdir -p ${DESTDIR}${PREFIX}/bin
 	cp -f dwm ${DESTDIR}${PREFIX}/bin
 	chmod 755 ${DESTDIR}${PREFIX}/bin/dwm
+	cp -f dwmswallow ${DESTDIR}${PREFIX}/bin
+	chmod 755 ${DESTDIR}${PREFIX}/bin/dwmswallow
 	mkdir -p ${DESTDIR}${MANPREFIX}/man1
 	sed "s/VERSION/${VERSION}/g" < dwm.1 > ${DESTDIR}${MANPREFIX}/man1/dwm.1
 	chmod 644 ${DESTDIR}${MANPREFIX}/man1/dwm.1
 
 uninstall:
 	rm -f ${DESTDIR}${PREFIX}/bin/dwm\
+		${DESTDIR}${MANPREFIX}/bin/dwmswallow\
 		${DESTDIR}${MANPREFIX}/man1/dwm.1
 
 .PHONY: all options clean dist install uninstall
diff --git a/config.def.h b/config.def.h
index 1c0b587..39e07a8 100644
--- a/config.def.h
+++ b/config.def.h
@@ -31,6 +31,11 @@ static const Rule rules[] = {
 	{ "Firefox",  NULL,       NULL,       1 << 8,       0,           -1 },
 };
 
+/* window swallowing */
+static const int swaldecay = 3;
+static const int swalretroactive = 1;
+static const char swalsymbol[] = "👅";
+
 /* layout(s) */
 static const float mfact     = 0.55; /* factor of master area size [0.05..0.95] */
 static const int nmaster     = 1;    /* number of clients in master area */
@@ -84,6 +89,7 @@ static Key keys[] = {
 	{ MODKEY,                       XK_period, focusmon,       {.i = +1 } },
 	{ MODKEY|ShiftMask,             XK_comma,  tagmon,         {.i = -1 } },
 	{ MODKEY|ShiftMask,             XK_period, tagmon,         {.i = +1 } },
+	{ MODKEY,                       XK_u,      swalstopsel,    {0} },
 	TAGKEYS(                        XK_1,                      0)
 	TAGKEYS(                        XK_2,                      1)
 	TAGKEYS(                        XK_3,                      2)
@@ -107,6 +113,7 @@ static Button buttons[] = {
 	{ ClkClientWin,         MODKEY,         Button1,        movemouse,      {0} },
 	{ ClkClientWin,         MODKEY,         Button2,        togglefloating, {0} },
 	{ ClkClientWin,         MODKEY,         Button3,        resizemouse,    {0} },
+	{ ClkClientWin,         MODKEY|ShiftMask, Button1,      swalmouse,      {0} },
 	{ ClkTagBar,            0,              Button1,        view,           {0} },
 	{ ClkTagBar,            0,              Button3,        toggleview,     {0} },
 	{ ClkTagBar,            MODKEY,         Button1,        tag,            {0} },
diff --git a/dwm.c b/dwm.c
index 664c527..390ef30 100644
--- a/dwm.c
+++ b/dwm.c
@@ -58,7 +58,7 @@
 #define TEXTW(X)                (drw_fontset_getwidth(drw, (X)) + lrpad)
 
 /* enums */
-enum { CurNormal, CurResize, CurMove, CurLast }; /* cursor */
+enum { CurNormal, CurResize, CurMove, CurSwal, CurLast }; /* cursor */
 enum { SchemeNorm, SchemeSel }; /* color schemes */
 enum { NetSupported, NetWMName, NetWMState, NetWMCheck,
        NetWMFullscreen, NetActiveWindow, NetWMWindowType,
@@ -66,6 +66,7 @@ enum { NetSupported, NetWMName, NetWMState, NetWMCheck,
 enum { WMProtocols, WMDelete, WMState, WMTakeFocus, WMLast }; /* default atoms */
 enum { ClkTagBar, ClkLtSymbol, ClkStatusText, ClkWinTitle,
        ClkClientWin, ClkRootWin, ClkLast }; /* clicks */
+enum { ClientRegular = 1, ClientSwallowee, ClientSwallower }; /* client types */
 
 typedef union {
 	int i;
@@ -95,6 +96,7 @@ struct Client {
 	int isfixed, isfloating, isurgent, neverfocus, oldstate, isfullscreen;
 	Client *next;
 	Client *snext;
+	Client *swallowedby;
 	Monitor *mon;
 	Window win;
 };
@@ -141,6 +143,28 @@ typedef struct {
 	int monitor;
 } Rule;
 
+typedef struct Swallow Swallow;
+struct Swallow {
+	/* Window class name, instance name (WM_CLASS) and title
+	 * (WM_NAME/_NET_WM_NAME, latter preferred if it exists). An empty string
+	 * implies a wildcard as per strstr(). */
+	char class[256];
+	char inst[256];
+	char title[256];
+
+	/* Used to delete swallow instance after 'swaldecay' windows were mapped
+	 * without the swallow having been consumed. 'decay' keeps track of the
+	 * remaining "charges". */
+	int decay;
+
+	/* The swallower, i.e. the client which will swallow the next mapped window
+	 * whose filters match the above properties. */
+	Client *client;
+
+	/* Linked list of registered swallow instances. */
+	Swallow *next;
+};
+
 /* function declarations */
 static void applyrules(Client *c);
 static int applysizehints(Client *c, int *x, int *y, int *w, int *h, int interact);
@@ -165,6 +189,7 @@ static void drawbar(Monitor *m);
 static void drawbars(void);
 static void enternotify(XEvent *e);
 static void expose(XEvent *e);
+static int fakesignal(void);
 static void focus(Client *c);
 static void focusin(XEvent *e);
 static void focusmon(const Arg *arg);
@@ -207,6 +232,16 @@ static void seturgent(Client *c, int urg);
 static void showhide(Client *c);
 static void sigchld(int unused);
 static void spawn(const Arg *arg);
+static void swal(Client *swer, Client *swee, int manage);
+static void swalreg(Client *c, const char* class, const char* inst, const char* title);
+static void swaldecayby(int decayby);
+static void swalmanage(Swallow *s, Window w, XWindowAttributes *wa);
+static Swallow *swalmatch(Window w);
+static void swalmouse(const Arg *arg);
+static void swalrm(Swallow *s);
+static void swalunreg(Client *c);
+static void swalstop(Client *c, Client *root);
+static void swalstopsel(const Arg *unused);
 static void tag(const Arg *arg);
 static void tagmon(const Arg *arg);
 static void tile(Monitor *);
@@ -229,6 +264,7 @@ static void updatewindowtype(Client *c);
 static void updatewmhints(Client *c);
 static void view(const Arg *arg);
 static Client *wintoclient(Window w);
+static int wintoclient2(Window w, Client **pc, Client **proot);
 static Monitor *wintomon(Window w);
 static int xerror(Display *dpy, XErrorEvent *ee);
 static int xerrordummy(Display *dpy, XErrorEvent *ee);
@@ -267,6 +303,7 @@ static Clr **scheme;
 static Display *dpy;
 static Drw *drw;
 static Monitor *mons, *selmon;
+static Swallow *swallows;
 static Window root, wmcheckwin;
 
 /* configuration, allows nested code to access above variables */
@@ -584,10 +621,12 @@ configurerequest(XEvent *e)
 	XConfigureRequestEvent *ev = &e->xconfigurerequest;
 	XWindowChanges wc;
 
-	if ((c = wintoclient(ev->window))) {
-		if (ev->value_mask & CWBorderWidth)
+	switch (wintoclient2(ev->window, &c, NULL)) {
+	case ClientRegular: /* fallthrough */
+	case ClientSwallowee:
+		if (ev->value_mask & CWBorderWidth) {
 			c->bw = ev->border_width;
-		else if (c->isfloating || !selmon->lt[selmon->sellt]->arrange) {
+		} else if (c->isfloating || !selmon->lt[selmon->sellt]->arrange) {
 			m = c->mon;
 			if (ev->value_mask & CWX) {
 				c->oldx = c->x;
@@ -615,7 +654,13 @@ configurerequest(XEvent *e)
 				XMoveResizeWindow(dpy, c->win, c->x, c->y, c->w, c->h);
 		} else
 			configure(c);
-	} else {
+		break;
+	case ClientSwallower:
+		/* Reject any move/resize requests for swallowers and communicate
+		 * refusal to client via a synthetic ConfigureNotify (ICCCM 4.1.5). */
+		configure(c);
+		break;
+	default:
 		wc.x = ev->x;
 		wc.y = ev->y;
 		wc.width = ev->width;
@@ -624,6 +669,7 @@ configurerequest(XEvent *e)
 		wc.sibling = ev->above;
 		wc.stack_mode = ev->detail;
 		XConfigureWindow(dpy, ev->window, ev->value_mask, &wc);
+		break;
 	}
 	XSync(dpy, False);
 }
@@ -648,11 +694,30 @@ createmon(void)
 void
 destroynotify(XEvent *e)
 {
-	Client *c;
+	Client *c, *swee, *root;
 	XDestroyWindowEvent *ev = &e->xdestroywindow;
 
-	if ((c = wintoclient(ev->window)))
+	switch (wintoclient2(ev->window, &c, &root)) {
+	case ClientRegular:
+		unmanage(c, 1);
+		break;
+	case ClientSwallowee:
+		swalstop(c, NULL);
 		unmanage(c, 1);
+		break;
+	case ClientSwallower:
+		/* If the swallower is swallowed by another client, terminate the
+		 * swallow. This cuts off the swallow chain after the client. */
+		swalstop(c, root);
+
+		/* Cut off the swallow chain before the client. */
+		for (swee = root; swee->swallowedby != c; swee = swee->swallowedby);
+		swee->swallowedby = NULL;
+
+		free(c);
+		updateclientlist();
+		break;
+	}
 }
 
 void
@@ -729,6 +794,12 @@ drawbar(Monitor *m)
 	drw_setscheme(drw, scheme[SchemeNorm]);
 	x = drw_text(drw, x, 0, w, bh, lrpad / 2, m->ltsymbol, 0);
 
+	/* Draw swalsymbol next to ltsymbol. */
+	if (m->sel && m->sel->swallowedby) {
+		w = TEXTW(swalsymbol);
+		x = drw_text(drw, x, 0, w, bh, lrpad / 2, swalsymbol, 0);
+	}
+
 	if ((w = m->ww - tw - x) > bh) {
 		if (m->sel) {
 			drw_setscheme(drw, scheme[m == selmon ? SchemeSel : SchemeNorm]);
@@ -781,6 +852,81 @@ expose(XEvent *e)
 		drawbar(m);
 }
 
+int
+fakesignal(void)
+{
+	/* Command syntax: <PREFIX><COMMAND>[<SEP><ARG>]... */
+	static const char sep[] = "###";
+	static const char prefix[] = "#!";
+
+	size_t numsegments, numargs;
+	char rootname[256];
+	char *segments[16] = {0};
+
+	/* Get root name, split by separator and find the prefix */
+	if (!gettextprop(root, XA_WM_NAME, rootname, sizeof(rootname))
+		|| strncmp(rootname, prefix, sizeof(prefix) - 1)) {
+		return 0;
+	}
+	numsegments = split(rootname + sizeof(prefix) - 1, sep, segments, sizeof(segments));
+	numargs = numsegments - 1; /* number of arguments to COMMAND */
+
+	if (!strcmp(segments[0], "swalreg")) {
+		/* Params: windowid, [class], [instance], [title] */
+		Window w;
+		Client *c;
+
+		if (numargs >= 1) {
+			w = strtoul(segments[1], NULL, 0);
+			switch (wintoclient2(w, &c, NULL)) {
+			case ClientRegular: /* fallthrough */
+			case ClientSwallowee:
+				swalreg(c, segments[2], segments[3], segments[4]);
+				break;
+			}
+		}
+	}
+	else if (!strcmp(segments[0], "swal")) {
+		/* Params: swallower's windowid, swallowee's window-id */
+		Client *swer, *swee;
+		Window winswer, winswee;
+		int typeswer, typeswee;
+
+		if (numargs >= 2) {
+			winswer = strtoul(segments[1], NULL, 0);
+			typeswer = wintoclient2(winswer, &swer, NULL);
+			winswee = strtoul(segments[2], NULL, 0);
+			typeswee = wintoclient2(winswee, &swee, NULL);
+			if ((typeswer == ClientRegular || typeswer == ClientSwallowee)
+				&& (typeswee == ClientRegular || typeswee == ClientSwallowee))
+				swal(swer, swee, 0);
+		}
+	}
+	else if (!strcmp(segments[0], "swalunreg")) {
+		/* Params: swallower's windowid */
+		Client *swer;
+		Window winswer;
+
+		if (numargs == 1) {
+			winswer = strtoul(segments[1], NULL, 0);
+			if ((swer = wintoclient(winswer)))
+				swalunreg(swer);
+		}
+	}
+	else if (!strcmp(segments[0], "swalstop")) {
+		/* Params: swallowee's windowid */
+		Client *swee;
+		Window winswee;
+
+		if (numargs == 1) {
+			winswee = strtoul(segments[1], NULL, 0);
+			if ((swee = wintoclient(winswee)))
+				swalstop(swee, NULL);
+		}
+	}
+	return 1;
+}
+
 void
 focus(Client *c)
 {
@@ -1090,15 +1236,37 @@ mappingnotify(XEvent *e)
 void
 maprequest(XEvent *e)
 {
+	Client *c, *swee, *root;
 	static XWindowAttributes wa;
 	XMapRequestEvent *ev = &e->xmaprequest;
+	Swallow *s;
 
 	if (!XGetWindowAttributes(dpy, ev->window, &wa))
 		return;
 	if (wa.override_redirect)
 		return;
-	if (!wintoclient(ev->window))
-		manage(ev->window, &wa);
+	switch (wintoclient2(ev->window, &c, &root)) {
+	case ClientRegular: /* fallthrough */
+	case ClientSwallowee:
+		/* Regulars and swallowees are always mapped. Nothing to do. */
+		break;
+	case ClientSwallower:
+		/* Remapping a swallower will simply stop the swallow. */
+		for (swee = root; swee->swallowedby != c; swee = swee->swallowedby);
+		swalstop(swee, root);
+		break;
+	default:
+		/* No client is managing the window. See if any swallows match. */
+		if ((s = swalmatch(ev->window)))
+			swalmanage(s, ev->window, &wa);
+		else
+			manage(ev->window, &wa);
+		break;
+	}
+
+	/* Reduce decay counter of all swallow instances. */
+	if (swaldecay)
+		swaldecayby(1);
 }
 
 void
@@ -1214,11 +1382,13 @@ propertynotify(XEvent *e)
 {
 	Client *c;
 	Window trans;
+	Swallow *s;
 	XPropertyEvent *ev = &e->xproperty;
 
-	if ((ev->window == root) && (ev->atom == XA_WM_NAME))
-		updatestatus();
-	else if (ev->state == PropertyDelete)
+	if ((ev->window == root) && (ev->atom == XA_WM_NAME)) {
+		if (!fakesignal())
+			updatestatus();
+	} else if (ev->state == PropertyDelete)
 		return; /* ignore */
 	else if ((c = wintoclient(ev->window))) {
 		switch(ev->atom) {
@@ -1240,6 +1410,9 @@ propertynotify(XEvent *e)
 			updatetitle(c);
 			if (c == c->mon->sel)
 				drawbar(c->mon);
+			if (swalretroactive && (s = swalmatch(c->win))) {
+				swal(s->client, c, 0);
+			}
 		}
 		if (ev->atom == netatom[NetWMWindowType])
 			updatewindowtype(c);
@@ -1567,6 +1740,7 @@ setup(void)
 	cursor[CurNormal] = drw_cur_create(drw, XC_left_ptr);
 	cursor[CurResize] = drw_cur_create(drw, XC_sizing);
 	cursor[CurMove] = drw_cur_create(drw, XC_fleur);
+	cursor[CurSwal] = drw_cur_create(drw, XC_bottom_side);
 	/* init appearance */
 	scheme = ecalloc(LENGTH(colors), sizeof(Clr *));
 	for (i = 0; i < LENGTH(colors); i++)
@@ -1653,6 +1827,331 @@ spawn(const Arg *arg)
 	}
 }
 
+/*
+ * Perform immediate swallow of client 'swee' by client 'swer'. 'manage' shall
+ * be set if swal() is called from swalmanage(). 'swer' and 'swee' must be
+ * regular or swallowee, but not swallower.
+ */
+void
+swal(Client *swer, Client *swee, int manage)
+{
+	Client *c, **pc;
+	int sweefocused = selmon->sel == swee;
+
+	/* Remove any swallows registered for the swer. Asking a swallower to
+	 * swallow another window is ambiguous and is thus avoided altogether. In
+	 * contrast, a swallowee can swallow in a well-defined manner by attaching
+	 * to the head of the swallow chain. */
+	if (!manage)
+		swalunreg(swer);
+
+	/* Disable fullscreen prior to swallow. Swallows involving fullscreen
+	 * windows produces quirky artefacts such as fullscreen terminals or tiled
+	 * pseudo-fullscreen windows. */
+	setfullscreen(swer, 0);
+	setfullscreen(swee, 0);
+
+	/* Swap swallowee into client and focus lists. Keeps current focus unless
+	 * the swer (which gets unmapped) is focused in which case the swee will
+	 * receive focus. */
+	detach(swee);
+	for (pc = &swer->mon->clients; *pc && *pc != swer; pc = &(*pc)->next);
+	*pc = swee;
+	swee->next = swer->next;
+	detachstack(swee);
+	for (pc = &swer->mon->stack; *pc && *pc != swer; pc = &(*pc)->snext);
+	*pc = swee;
+	swee->snext = swer->snext;
+	swee->mon = swer->mon;
+	if (sweefocused) {
+		detachstack(swee);
+		attachstack(swee);
+		selmon = swer->mon;
+	}
+	swee->tags = swer->tags;
+	swee->isfloating = swer->isfloating;
+	for (c = swee; c->swallowedby; c = c->swallowedby);
+	c->swallowedby = swer;
+
+	/* Configure geometry params obtained from patches (e.g. cfacts) here. */
+	// swee->cfact = swer->cfact;
+
+	/* ICCCM 4.1.3.1 */
+	setclientstate(swer, WithdrawnState);
+	if (manage)
+		setclientstate(swee, NormalState);
+
+	if (swee->isfloating || !swee->mon->lt[swee->mon->sellt]->arrange)
+		XRaiseWindow(dpy, swee->win);
+	resize(swee, swer->x, swer->y, swer->w, swer->h, 0);
+
+	focus(NULL);
+	arrange(NULL);
+	if (manage)
+		XMapWindow(dpy, swee->win);
+	XUnmapWindow(dpy, swer->win);
+	restack(swer->mon);
+}
+
+/*
+ * Register a future swallow with swallower. 'c' 'class', 'inst' and 'title'
+ * shall point null-terminated strings or be NULL, implying a wildcard. If an
+ * already existing swallow instance targets 'c' its filters are updated and no
+ * new swallow instance is created. 'c' may be ClientRegular or ClientSwallowee.
+ * Complement to swalrm().
+ */
+void swalreg(Client *c, const char *class, const char *inst, const char *title)
+{
+	Swallow *s;
+
+	if (!c)
+		return;
+
+	for (s = swallows; s; s = s->next) {
+		if (s->client == c) {
+			if (class)
+				strncpy(s->class, class, sizeof(s->class) - 1);
+			else
+				s->class[0] = '\0';
+			if (inst)
+				strncpy(s->inst, inst, sizeof(s->inst) - 1);
+			else
+				s->inst[0] = '\0';
+			if (title)
+				strncpy(s->title, title, sizeof(s->title) - 1);
+			else
+				s->title[0] = '\0';
+			s->decay = swaldecay;
+
+			/* Only one swallow per client. May return after first hit. */
+			return;
+		}
+	}
+
+	s = ecalloc(1, sizeof(Swallow));
+	s->decay = swaldecay;
+	s->client = c;
+	if (class)
+		strncpy(s->class, class, sizeof(s->class) - 1);
+	if (inst)
+		strncpy(s->inst, inst, sizeof(s->inst) - 1);
+	if (title)
+		strncpy(s->title, title, sizeof(s->title) - 1);
+
+	s->next = swallows;
+	swallows = s;
+}
+
+/*
+ * Decrease decay counter of all registered swallows by 'decayby' and remove any
+ * swallow instances whose counter is less than or equal to zero.
+ */
+void
+swaldecayby(int decayby)
+{
+	Swallow *s, *t;
+
+	for (s = swallows; s; s = t) {
+		s->decay -= decayby;
+		t = s->next;
+		if (s->decay <= 0)
+			swalrm(s);
+	}
+}
+
+/*
+ * Window configuration and client setup for new windows which are to be
+ * swallowed immediately. Pendant to manage() for such windows.
+ */
+void
+swalmanage(Swallow *s, Window w, XWindowAttributes *wa)
+{
+	Client *swee, *swer;
+	XWindowChanges wc;
+
+	swer = s->client;
+	swalrm(s);
+
+	/* Perform bare minimum setup of a client for window 'w' such that swal()
+	 * may be used to perform the swallow. The following lines are basically a
+	 * minimal implementation of manage() with a few chunks delegated to
+	 * swal(). */
+	swee = ecalloc(1, sizeof(Client));
+	swee->win = w;
+	swee->mon = swer->mon;
+	swee->oldbw = wa->border_width;
+	swee->bw = borderpx;
+	attach(swee);
+	attachstack(swee);
+	updatetitle(swee);
+	updatesizehints(swee);
+	XSelectInput(dpy, swee->win, EnterWindowMask|FocusChangeMask|PropertyChangeMask|StructureNotifyMask);
+	wc.border_width = swee->bw;
+	XConfigureWindow(dpy, swee->win, CWBorderWidth, &wc);
+	grabbuttons(swee, 0);
+	XChangeProperty(dpy, root, netatom[NetClientList], XA_WINDOW, 32, PropModeAppend,
+		(unsigned char *) &(swee->win), 1);
+
+	swal(swer, swee, 1);
+}
+
+/*
+ * Return swallow instance which targets window 'w' as determined by its class
+ * name, instance name and window title. Returns NULL if none is found. Pendant
+ * to wintoclient().
+ */
+Swallow *
+swalmatch(Window w)
+{
+	XClassHint ch = { NULL, NULL };
+	Swallow *s = NULL;
+	char title[sizeof(s->title)];
+
+	XGetClassHint(dpy, w, &ch);
+	if (!gettextprop(w, netatom[NetWMName], title, sizeof(title)))
+		gettextprop(w, XA_WM_NAME, title, sizeof(title));
+
+	for (s = swallows; s; s = s->next) {
+		if ((!ch.res_class || strstr(ch.res_class, s->class))
+			&& (!ch.res_name || strstr(ch.res_name, s->inst))
+			&& (title[0] == '\0' || strstr(title, s->title)))
+			break;
+	}
+
+	if (ch.res_class)
+		XFree(ch.res_class);
+	if (ch.res_name)
+		XFree(ch.res_name);
+	return s;
+}
+
+/*
+ * Interactive drag-and-drop swallow.
+ */
+void
+swalmouse(const Arg *arg)
+{
+	Client *swer, *swee;
+	XEvent ev;
+
+	if (!(swee = selmon->sel))
+		return;
+
+	if (XGrabPointer(dpy, root, False, ButtonPressMask|ButtonReleaseMask, GrabModeAsync,
+		GrabModeAsync, None, cursor[CurSwal]->cursor, CurrentTime) != GrabSuccess)
+		return;
+
+	do {
+		XMaskEvent(dpy, MOUSEMASK|ExposureMask|SubstructureRedirectMask, &ev);
+		switch(ev.type) {
+		case ConfigureRequest: /* fallthrough */
+		case Expose: /* fallthrough */
+		case MapRequest:
+			handler[ev.type](&ev);
+			break;
+		}
+	} while (ev.type != ButtonRelease);
+	XUngrabPointer(dpy, CurrentTime);
+
+	if ((swer = wintoclient(ev.xbutton.subwindow))
+		&& swer != swee)
+		swal(swer, swee, 0);
+
+	/* Remove accumulated pending EnterWindow events caused by the mouse
+	 * movements. */
+	XCheckMaskEvent(dpy, EnterWindowMask, &ev);
+}
+
+/*
+ * Delete swallow instance swallows and free its resources. Complement to
+ * swalreg(). If NULL is passed all swallows are deleted from.
+ */
+void
+swalrm(Swallow *s)
+{
+	Swallow *t, **ps;
+
+	if (s) {
+		for (ps = &swallows; *ps && *ps != s; ps = &(*ps)->next);
+		*ps = s->next;
+		free(s);
+	}
+	else {
+		for(s = swallows; s; s = t) {
+			t = s->next;
+			free(s);
+		}
+		swallows = NULL;
+	}
+}
+
+/*
+ * Removes swallow instance targeting 'c' if it exists. Complement to swalreg().
+ */
+void swalunreg(Client *c) { Swallow *s;
+
+	for (s = swallows; s; s = s->next) {
+		if (c == s->client) {
+			swalrm(s);
+			/* Max. 1 registered swallow per client. No need to continue. */
+			break;
+		}
+	}
+}
+
+/*
+ * Stop an active swallow of swallowed client 'swee' and remap the swallower.
+ * If 'swee' is a swallower itself 'root' must point the root client of the
+ * swallow chain containing 'swee'.
+ */
+void
+swalstop(Client *swee, Client *root)
+{
+	Client *swer;
+
+	if (!swee || !(swer = swee->swallowedby))
+		return;
+
+	swee->swallowedby = NULL;
+	root = root ? root : swee;
+	swer->mon = root->mon;
+	swer->tags = root->tags;
+	swer->next = root->next;
+	root->next = swer;
+	swer->snext = root->snext;
+	root->snext = swer;
+	swer->isfloating = swee->isfloating;
+
+	/* Configure geometry params obtained from patches (e.g. cfacts) here. */
+	// swer->cfact = 1.0;
+
+	/* If swer is not in tiling mode reuse swee's geometry. */
+	if (swer->isfloating || !root->mon->lt[root->mon->sellt]->arrange) {
+		XRaiseWindow(dpy, swer->win);
+		resize(swer, swee->x, swee->y, swee->w, swee->h, 0);
+	}
+
+	/* Override swer's border scheme which may be using SchemeSel. */
+	XSetWindowBorder(dpy, swer->win, scheme[SchemeNorm][ColBorder].pixel);
+
+	/* ICCCM 4.1.3.1 */
+	setclientstate(swer, NormalState);
+
+	XMapWindow(dpy, swer->win);
+	focus(NULL);
+	arrange(swer->mon);
+}
+
+/*
+ * Stop active swallow for currently selected client.
+ */
+void
+swalstopsel(const Arg *unused)
+{
+	if (selmon->sel)
+		swalstop(selmon->sel, NULL);
+}
+
 void
 tag(const Arg *arg)
 {
@@ -1768,6 +2267,9 @@ unmanage(Client *c, int destroyed)
 	Monitor *m = c->mon;
 	XWindowChanges wc;
 
+	/* Remove all swallow instances targeting client. */
+	swalunreg(c);
+
 	detach(c);
 	detachstack(c);
 	if (!destroyed) {
@@ -1790,14 +2292,27 @@ unmanage(Client *c, int destroyed)
 void
 unmapnotify(XEvent *e)
 {
+
 	Client *c;
 	XUnmapEvent *ev = &e->xunmap;
+	int type;
 
-	if ((c = wintoclient(ev->window))) {
-		if (ev->send_event)
-			setclientstate(c, WithdrawnState);
-		else
-			unmanage(c, 0);
+	type = wintoclient2(ev->window, &c, NULL);
+	if (type && ev->send_event) {
+		setclientstate(c, WithdrawnState);
+		return;
+	}
+	switch (type) {
+	case ClientRegular:
+		unmanage(c, 0);
+		break;
+	case ClientSwallowee:
+		swalstop(c, NULL);
+		unmanage(c, 0);
+		break;
+	case ClientSwallower:
+		/* Swallowers are never mapped. Nothing to do. */
+		break;
 	}
 }
 
@@ -1839,15 +2354,19 @@ updatebarpos(Monitor *m)
 void
 updateclientlist()
 {
-	Client *c;
+	Client *c, *d;
 	Monitor *m;
 
 	XDeleteProperty(dpy, root, netatom[NetClientList]);
-	for (m = mons; m; m = m->next)
-		for (c = m->clients; c; c = c->next)
-			XChangeProperty(dpy, root, netatom[NetClientList],
-				XA_WINDOW, 32, PropModeAppend,
-				(unsigned char *) &(c->win), 1);
+	for (m = mons; m; m = m->next) {
+		for (c = m->clients; c; c = c->next) {
+			for (d = c; d; d = d->swallowedby) {
+				XChangeProperty(dpy, root, netatom[NetClientList],
+					XA_WINDOW, 32, PropModeAppend,
+					(unsigned char *) &(c->win), 1);
+			}
+		}
+	}
 }
 
 int
@@ -2060,6 +2579,43 @@ wintoclient(Window w)
 	return NULL;
 }
 
+/*
+ * Writes client managing window 'w' into 'pc' and returns type of client. If
+ * no client is found NULL is written to 'pc' and zero is returned. If a client
+ * is found and is a swallower (ClientSwallower) and proot is not NULL the root
+ * client of the swallow chain is written to 'proot'.
+ */
+int
+wintoclient2(Window w, Client **pc, Client **proot)
+{
+	Monitor *m;
+	Client *c, *d;
+
+	for (m = mons; m; m = m->next) {
+		for (c = m->clients; c; c = c->next) {
+			if (c->win == w) {
+				*pc = c;
+				if (c->swallowedby)
+					return ClientSwallowee;
+				else
+					return ClientRegular;
+			}
+			else {
+				for (d = c->swallowedby; d; d = d->swallowedby) {
+					if (d->win == w) {
+						if (proot)
+							*proot = c;
+						*pc = d;
+						return ClientSwallower;
+					}
+				}
+			}
+		}
+	}
+	*pc = NULL;
+	return 0;
+}
+
 Monitor *
 wintomon(Window w)
 {
diff --git a/dwmswallow b/dwmswallow
new file mode 100755
index 0000000..9400606
--- /dev/null
+++ b/dwmswallow
@@ -0,0 +1,120 @@
+#!/usr/bin/env sh
+
+# Separator and command prefix, as defined in dwm.c:fakesignal()
+SEP='###'
+PREFIX='#!'
+
+# Asserts that all arguments are valid X11 window IDs, i.e. positive integers.
+# For the purpose of this script 0 is declared invalid aswe
+is_winid() {
+	while :; do
+		# Given input incompatible to %d, some implementations of printf return
+		# an error while others silently evaluate the expression to 0.
+		if ! wid=$(printf '%d' "$1" 2>/dev/null) || [ "$wid" -le 0 ]; then
+			return 1
+		fi
+
+		[ -n "$2" ] && shift || break
+	done
+}
+
+# Prints usage help. If "$1" is provided, function exits script after
+# execution.
+usage() {
+	[ -t 1 ] && myprintf=printf || myprintf=true
+	msg="$(cat <<-EOF
+	dwm window swallowing command-line interface. Usage:
+
+	  $($myprintf "\033[1m")dwmswallow $($myprintf "\033[3m")SWALLOWER [-c CLASS] [-i INSTANCE] [-t TITLE]$($myprintf "\033[0m")
+	    Register window $($myprintf "\033[3m")SWALLOWER$($myprintf "\033[0m") to swallow the next future window whose attributes
+	    match the $($myprintf "\033[3m")CLASS$($myprintf "\033[0m") name, $($myprintf "\033[3m")INSTANCE$($myprintf "\033[0m") name and window $($myprintf "\033[3m")TITLE$($myprintf "\033[0m") filters using basic
+	    string-matching. An omitted filter will match anything.
+
+	  $($myprintf "\033[1m")dwmswallow $($myprintf "\033[3m")SWALLOWER -d$($myprintf "\033[0m")
+	    Deregister queued swallow for window $($myprintf "\033[3m")SWALLOWER$($myprintf "\033[0m"). Inverse of above signature.
+
+	  $($myprintf "\033[1m")dwmswallow $($myprintf "\033[3m")SWALLOWER SWALLOWEE$($myprintf "\033[0m")
+	    Perform immediate swallow of window $($myprintf "\033[3m")SWALLOWEE$($myprintf "\033[0m") by window $($myprintf "\033[3m")SWALLOWER$($myprintf "\033[0m").
+
+	  $($myprintf "\033[1m")dwmswallow $($myprintf "\033[3m")SWALLOWEE -s$($myprintf "\033[0m")
+	    Stop swallow of window $($myprintf "\033[3m")SWALLOWEE$($myprintf "\033[0m"). Inverse of the above signature. Visible
+	    windows only.
+
+	  $($myprintf "\033[1m")dwmswallow -h$($myprintf "\033[0m")
+	    Show this usage information.
+	EOF
+	)"
+
+	if [ -n "$1" ]; then
+		echo "$msg" >&2
+		exit "$1"
+	else
+		echo "$msg"
+	fi
+}
+
+# Determine number of leading positional arguments
+arg1="$1" # save for later
+arg2="$2" # save for later
+num_pargs=0
+while :; do
+	case "$1" in
+	-*|"") break ;;
+	*) num_pargs=$((num_pargs + 1)); shift ;;
+	esac
+done
+
+case "$num_pargs" in
+1)
+	! is_winid "$arg1" && usage 1
+
+	widswer="$arg1"
+	if [ "$1" = "-d" ] && [ "$#" -eq 1 ]; then
+		if name="$(printf "${PREFIX}swalunreg${SEP}%u" "$widswer" 2>/dev/null)"; then
+			xsetroot -name "$name"
+		else
+			usage 1
+		fi
+	elif [ "$1" = "-s" ] && [ "$#" -eq 1 ]; then
+		widswee="$arg1"
+		if name="$(printf "${PREFIX}swalstop${SEP}%u" "$widswee" 2>/dev/null)"; then
+			xsetroot -name "$name"
+		else
+			usage 1
+		fi
+	else
+		while :; do
+			case "$1" in
+			-c) [ -n "$2" ] && { class="$2"; shift 2; } || usage 1 ;;
+			-i) [ -n "$2" ] && { instance="$2"; shift 2; } || usage 1 ;;
+			-t) [ -n "$2" ] && { title="$2"; shift 2; } || usage 1 ;;
+			"") break ;;
+			*)	usage 1 ;;
+			esac
+		done
+		widswer="$arg1"
+		if name="$(printf "${PREFIX}swalreg${SEP}%u${SEP}%s${SEP}%s${SEP}%s" "$widswer" "$class" "$instance" "$title" 2>/dev/null)"; then
+			xsetroot -name "$name"
+		else
+			usage 1
+		fi
+	fi
+	;;
+2)
+	! is_winid "$arg1" "$arg2" || [ -n "$1" ] && usage 1
+
+	widswer="$arg1"
+	widswee="$arg2"
+	if name="$(printf "${PREFIX}swal${SEP}%u${SEP}%u" "$widswer" "$widswee" 2>/dev/null)"; then
+		xsetroot -name "$name"
+	else
+		usage 1
+	fi
+	;;
+*)
+	if [ "$arg1" = "-h" ] && [ $# -eq 1 ]; then
+		usage
+	else
+		usage 1
+	fi
+esac
diff --git a/util.c b/util.c
index fe044fc..1f51877 100644
--- a/util.c
+++ b/util.c
@@ -33,3 +33,33 @@ die(const char *fmt, ...) {
 
 	exit(1);
 }
+
+/*
+ * Splits a string into segments according to a separator. A '\0' is written to
+ * the end of every segment. The beginning of every segment is written to
+ * 'pbegin'. Only the first 'maxcount' segments will be written if
+ * maxcount > 0. Inspired by python's split.
+ *
+ * Used exclusively by fakesignal() to split arguments.
+ */
+size_t
+split(char *s, const char* sep, char **pbegin, size_t maxcount) {
+
+	char *p, *q;
+	const size_t seplen = strlen(sep);
+	size_t count = 0;
+
+	maxcount = maxcount == 0 ? (size_t)-1 : maxcount;
+	p = s;
+	while ((q = strstr(p, sep)) != NULL && count < maxcount) {
+		pbegin[count] = p;
+		*q = '\0';
+		p = q + seplen;
+		count++;
+	}
+	if (count < maxcount) {
+		pbegin[count] = p;
+		count++;
+	}
+	return count;
+}
diff --git a/util.h b/util.h
index f633b51..670345f 100644
--- a/util.h
+++ b/util.h
@@ -6,3 +6,4 @@
 
 void die(const char *fmt, ...);
 void *ecalloc(size_t nmemb, size_t size);
+size_t split(char *s, const char* sep, char **pbegin, size_t maxcount);
-- 
2.25.1

