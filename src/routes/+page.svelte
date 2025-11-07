<script lang="ts">
	import { page } from '$app/stores';
	import CalendarHeader from '$lib/components/CalendarHeader.svelte';
	import DataTransfers from '$lib/components/dataTransfers.svelte';
	import { tick } from 'svelte';
	import { calcNDay } from '$lib/averages.ts';
	import { changeCalendarMonth, getFirstDayOfMonth, toISODateString } from '$lib/dateHelpers.ts';
	import { updateDateQuery } from '$lib/updateDate.ts';
	import { calorieDateMap } from '$/stores.ts';
	import '$/styles/calendar.css';

	const notEnoughData = 'Not enough data';

	let selectedDate = '';
	let calendarMonth = '';
	let calorieInput: HTMLInputElement;
	let showResetModal = false;
	let resetSuccess = false;
	let dateInputValue = '';

	let startX: number;
	let endX: number;

	$: calendarDates = generateCalendarDates(calendarMonth, $calorieDateMap);

	$: if ($calorieDateMap[selectedDate] === undefined) {
		calorieDateMap.update((map) => {
			return { ...map, [selectedDate]: null };
		});
	}

	page.subscribe(($page) => {
		const date = $page.url.searchParams.get('date');
		selectedDate = date ?? new Date().toDateString();

		if (!calendarMonth) {
			calendarMonth = getFirstDayOfMonth(selectedDate);
		}
	});

	function handleMonthNavigation(direction: 'next' | 'previous') {
		calendarMonth = changeCalendarMonth(calendarMonth, direction);
	}

	function handleMonthPicker(monthValue: string) {
		const [year, month] = monthValue.split('-').map(Number);
		calendarMonth = new Date(year, month - 1, 1).toDateString();
	}

	// Initialize dateInputValue with the current pathname in ISO format
	$: dateInputValue = toISODateString(selectedDate);

	function generateCalendarDates(
		currentMonthString: string,
		calorieMap: Record<string, number | null>
	) {
		const currentDate = new Date(currentMonthString);
		const year = currentDate.getFullYear();
		const month = currentDate.getMonth();

		const firstDayOfMonth = new Date(year, month, 1);
		const lastDayOfMonth = new Date(year, month + 1, 0);
		const startingDayOfWeek = firstDayOfMonth.getDay();

		const dates = [];
		const today = new Date().toDateString();

		// Add days from previous month to fill the grid
		for (let i = startingDayOfWeek - 1; i >= 0; i--) {
			const date = new Date(year, month, -i);
			const dateString = date.toDateString();
			dates.push({
				day: date.getDate(),
				dateString,
				isCurrentMonth: false,
				isToday: dateString === today,
				calories: calorieMap[dateString]
			});
		}

		// Add all days of current month
		for (let day = 1; day <= lastDayOfMonth.getDate(); day++) {
			const date = new Date(year, month, day);
			const dateString = date.toDateString();
			dates.push({
				day: day,
				dateString,
				isCurrentMonth: true,
				isToday: dateString === today,
				calories: calorieMap[dateString]
			});
		}

		// Add days from next month to complete the grid
		const totalCells = Math.ceil(dates.length / 7) * 7;
		for (let day = 1; dates.length < totalCells; day++) {
			const date = new Date(year, month + 1, day);
			const dateString = date.toDateString();
			dates.push({
				day: day,
				dateString,
				isCurrentMonth: false,
				isToday: dateString === today,
				calories: calorieMap[dateString]
			});
		}

		return dates;
	}

	async function handleCalendarDateClick(dateString: string) {
		updateDateQuery(dateString);
		await tick();
		setTimeout(() => {
			if (calorieInput) {
				calorieInput.focus();
				calorieInput.select();
			}
		}, 10);
	}

	function handleSwipe() {
		if (startX - endX > 50) {
			handleMonthNavigation('next');
		} else if (endX - startX > 50) {
			handleMonthNavigation('previous');
		}
	}

	function handleTouchStart(event: TouchEvent) {
		startX = event.touches[0].clientX;
	}

	function handleTouchEnd(event: TouchEvent) {
		endX = event.changedTouches[0].clientX;
		handleSwipe();
	}

	// If the date query param is somehow invalid, try to set it to today
	if (
		!selectedDate ||
		typeof selectedDate !== 'string' ||
		isNaN(new Date(selectedDate).valueOf())
	) {
		updateDateQuery(new Date().toDateString());
	}

	function handleSaveAndNext() {
		updateDateQuery(selectedDate, 'nextDay');

		setTimeout(() => {
			if (calorieInput) {
				calorieInput.focus();
				calorieInput.select();
			}
		}, 100);
	}

	function closeModal() {
		showResetModal = false;
	}

	function confirmReset() {
		calorieDateMap.reset();
		showResetModal = false;
		resetSuccess = true;

		setTimeout(() => {
			resetSuccess = false;
		}, 3000);
	}

	function handleKeydown(event: KeyboardEvent) {
		if (event.key === 'Escape' && showResetModal) {
			closeModal();
		}
	}
</script>

<svelte:head>
	<title>Rolling Calorie Averages</title>
	<meta
		name="description"
		content="Get rolling averages of your calorie intake for the last 3 days, week, or month."
	/>
</svelte:head>
<svelte:window on:keydown={handleKeydown} />

<section class="calendar" on:touchstart={handleTouchStart} on:touchend={handleTouchEnd}>
	<CalendarHeader
		{selectedDate}
		{calendarMonth}
		onMonthNavigation={handleMonthNavigation}
		onMonthPicker={handleMonthPicker}
	/>
	<div class="calendar-grid">
		<div class="calendar-header">
			<div class="day-name">Sun</div>
			<div class="day-name">Sat</div>
		</div>
		{#each calendarDates as date}
			<div
				class="calendar-date {date.isCurrentMonth ? 'current-month' : 'other-month'} {date.isToday
					? 'today'
					: ''} {date.dateString === selectedDate ? 'selected' : ''}"
				on:click={() => handleCalendarDateClick(date.dateString)}
				role="button"
				tabindex="0"
				on:keydown={(e) => e.key === 'Enter' && handleCalendarDateClick(date.dateString)}
			>
				<div class="date-number">{date.day}</div>
				{#if date.calories !== undefined && date.calories !== null}
					<div class="calories-display">{date.calories}</div>
				{/if}
			</div>
		{/each}
	</div>
</section>

<section class="todays-calories">
	<div class="calorie-input-container">
		<label for="calories">Calories for {selectedDate}</label>
		<div class="input-group">
			<input
				id="calories"
				type="number"
				placeholder="Enter calories..."
				bind:value={$calorieDateMap[selectedDate]}
				bind:this={calorieInput}
				on:keydown={(e) => e.key === 'Enter' && handleSaveAndNext()}
			/>
			<button class="save-button" on:click={handleSaveAndNext}> Save & Next → </button>
		</div>
		{#if $calorieDateMap[selectedDate]}
			<div class="calorie-feedback">
				✓ {$calorieDateMap[selectedDate]} calories recorded
			</div>
		{/if}
	</div>
</section>

<section class="calorie-averages">
	<div class="averages-container">
		<h3 class="averages-title">Rolling Averages</h3>
		<div class="averages-grid">
			<div class="average-card">
				<div class="average-period">
					<span class="period-icon">📊</span>
					<span class="period-text">3 Days</span>
				</div>
				<div class="average-value">
					{calcNDay(selectedDate, 3, $calorieDateMap) || notEnoughData}
					{#if calcNDay(selectedDate, 3, $calorieDateMap)}
						<span class="value-unit">cal/day</span>
					{/if}
				</div>
			</div>

			<div class="average-card">
				<div class="average-period">
					<span class="period-icon">📈</span>
					<span class="period-text">5 Days</span>
				</div>
				<div class="average-value">
					{calcNDay(selectedDate, 5, $calorieDateMap) || notEnoughData}
					{#if calcNDay(selectedDate, 5, $calorieDateMap)}
						<span class="value-unit">cal/day</span>
					{/if}
				</div>
			</div>

			<div class="average-card">
				<div class="average-period">
					<span class="period-icon">📅</span>
					<span class="period-text">2 Weeks</span>
				</div>
				<div class="average-value">
					{calcNDay(selectedDate, 14, $calorieDateMap) || notEnoughData}
					{#if calcNDay(selectedDate, 14, $calorieDateMap)}
						<span class="value-unit">cal/day</span>
					{/if}
				</div>
			</div>

			<div class="average-card">
				<div class="average-period">
					<span class="period-icon">🗓️</span>
					<span class="period-text">1 Month</span>
				</div>
				<div class="average-value">
					{calcNDay(selectedDate, 30, $calorieDateMap) || notEnoughData}
					{#if calcNDay(selectedDate, 30, $calorieDateMap)}
						<span class="value-unit">cal/day</span>
					{/if}
				</div>
			</div>
		</div>

		<!-- Add some helpful context -->
		<div class="averages-info">
			<p>💡 Rolling averages help smooth out daily fluctuations and show trends over time.</p>
		</div>
	</div>
</section>
<DataTransfers />

<section class="reset-section">
	<button class="reset-trigger-button" on:click={() => (showResetModal = true)}>
		🗑️ Reset All Data
	</button>

	{#if resetSuccess}
		<div class="reset-success-message">✅ All calorie data has been successfully reset!</div>
	{/if}
</section>

<!-- Reset modal -->
{#if showResetModal}
	<div class="modal-backdrop" on:click={closeModal} role="dialog" aria-modal="true">
		<div class="modal-content" on:click|stopPropagation>
			<div class="modal-header">
				<h3>⚠️ Confirm Data Reset</h3>
				<button class="modal-close" on:click={closeModal} aria-label="Close modal"> ✕ </button>
			</div>

			<div class="modal-body">
				<p>Are you sure you want to permanently delete all your calorie recordings?</p>
				<p class="warning-text">
					This action cannot be undone and will remove all historical data.
				</p>

				<div class="data-summary">
					<strong
						>You currently have data for {Object.keys($calorieDateMap).filter(
							(key) => $calorieDateMap[key] !== null
						).length} days</strong
					>
				</div>
			</div>

			<div class="modal-footer">
				<button class="cancel-button" on:click={closeModal}> Cancel </button>
				<button class="confirm-reset-button" on:click={confirmReset}> Yes, Reset All Data </button>
			</div>
			Í
		</div>
	</div>
{/if}
