---
layout: default
title: Tao of Programming
---

# Countdowns

<div id="countdowns-wrapper">
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/zepto/1.1.6/zepto.min.js"></script>
<script>
"use strict";

var countdowns = [
	{
		date: new Date("February 11 2016 00:00:01 GMT+02:00"),
	},
	{
		date: new Date("October 17 2016 23:00:01 GMT+02:00"),
	},
];

countdowns.sort();

var pluralize = function (count, str) {
	return count == 1 ? str : (str + 's');
};

var getRemainingTime = function (now, then, pastString) {
	var multis = [
		['day', 24 * 60 * 60],
		['hour', 60 * 60],
		['minute', 60],
		['second', 1],
	];

	var parts = [];
	var remainingSeconds = (then - now) / 1000;
	if (remainingSeconds < 1) {
		return pastString;
	}

	for (var i = 0; i < multis.length; i++) {
		var m = multis[i];

		var count = Math.floor(remainingSeconds / m[1]);
		remainingSeconds = remainingSeconds % m[1];

		if (count) {
			parts.push(count + ' ' + pluralize(count, m[0]));
		}
	}

	return parts.join(', ');
}

var updateTimers = function () {
	$.each(countdowns, function (index, item) {
		item.$timer.text(getRemainingTime(Date.now(), item.date.getTime(), "Happened!"));
	});
};


$(function() {
	var $countdownsWrapper = $("#countdowns-wrapper");

	countdowns = $.map(countdowns, function (countdown) {
		var $timer = $('<div>').addClass('timer');
		var $el = $('<div>')
			.addClass('countdown')
			.append($timer)
			.append($('<div>').addClass('date').text('until ' + countdown.date));
		$countdownsWrapper.append($el);
		return $.extend(countdown, { $timer: $timer });
	});

	updateTimers();
	setInterval(updateTimers, 1000);
});


</script>

<style>
.countdown {
	margin-top: 1.5em;
	font-size: 1.6em;
}

.countdown .date {
	color: #666;
	font-size: 65%;
}
</style>
