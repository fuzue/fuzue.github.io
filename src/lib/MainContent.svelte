<script lang="ts">
	import ChangingLogo from '$lib/ChangingLogo.svelte';
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
	import { goto } from '$app/navigation';
	import { page } from '$app/stores';

	export let initialSection: number = 0;

	let changingLogo: ChangingLogo;
	let randomNum = Math.floor(Math.random() * 360);
	let scrollY = 0;
	let innerHeight = 0;
	let currentStep = initialSection;

	const steps = [
		{ id: 'home', title: 'Welcome to Fuzuê', bgColor: 'white', textColor: 'text-slate-950', url: '/' },
		{ id: 'about', title: 'About Us', content: 'Lorem ipsum dolor sit amet, consectetur adipiscing elit. Sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.', bgColor: 'black', textColor: 'text-white', url: '/about' },
		{ 
			id: 'projects', 
			title: 'Our Services', 
			content: 'Explore our portfolio of innovative projects and solutions.', 
			bgColor: 'white', 
			textColor: 'text-slate-950',
			url: '/projects',
			hasHorizontalScroll: true,
			projects: [
				{ title: 'E-Commerce Platform', description: 'Modern online shopping experience with advanced features and seamless user interface.', tech: 'React, Node.js, MongoDB', bgColor: 'bg-blue-100' },
				{ title: 'Mobile Banking App', description: 'Secure and intuitive mobile banking solution with real-time transactions and analytics.', tech: 'React Native, Express, PostgreSQL', bgColor: 'bg-green-100' },
				{ title: 'AI-Powered Analytics', description: 'Intelligent data analysis platform using machine learning for business insights.', tech: 'Python, TensorFlow, AWS', bgColor: 'bg-purple-100' },
				{ title: 'IoT Smart Home', description: 'Connected home automation system with voice control and energy management.', tech: 'Arduino, MQTT, Vue.js', bgColor: 'bg-orange-100' },
				{ title: 'Blockchain Wallet', description: 'Secure cryptocurrency wallet with multi-chain support and DeFi integration.', tech: 'Solidity, Web3.js, Next.js', bgColor: 'bg-pink-100' }
			]
		},
		{ id: 'contact', title: 'Contact', content: 'Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur.', email: 'contact@fuzue.tech', bgColor: 'black', textColor: 'text-white', url: '/contact' }
	];

	// Project navigation for services section
	let currentProjectIndex = 0;
	let isNavigatingProgrammatically = false;
	let isInitialLoad = true;
	
	function nextProject(totalProjects: number) {
		currentProjectIndex = Math.min(currentProjectIndex + 1, totalProjects - 1);
	}
	
	function prevProject() {
		currentProjectIndex = Math.max(currentProjectIndex - 1, 0);
	}
	
	// URL and routing management  
	function updateURL(stepIndex: number) {
		if (browser && !isNavigatingProgrammatically && !isInitialLoad) {
			const step = steps[stepIndex];
			if (step && step.url !== $page.url.pathname) {
				goto(step.url, { replaceState: true, noScroll: true });
			}
		}
	}
	
	// Calculate scroll progress for smooth animations
	let scrollProgress = 0;
	let logoSize = 384; // Initial size in pixels (w-96 = 384px)
	let logoTopPosition = 50; // Initial position as percentage
	let headerHeight = 0; // Height of the expanding white background
	let isTransitioning = false; // Track if we're in logo transition
	let visibleSections = new Set(); // Track which sections are visible
	
	// Scroll hint arrow
	let showScrollHint = false;
	let hasUserScrolled = false;
	let scrollHintTimer: number;
	
	function startScrollHintTimer() {
		if (!hasUserScrolled && scrollY === 0) {
			scrollHintTimer = setTimeout(() => {
				if (!hasUserScrolled && scrollY === 0) {
					showScrollHint = true;
				}
			}, 5000);
		}
	}
	
	function hideScrollHint() {
		if (scrollY > 0 && !hasUserScrolled) {
			hasUserScrolled = true;
			showScrollHint = false;
			clearTimeout(scrollHintTimer);
		}
	}
	
	// Navigation indicator animation
	let navIndicatorLeft = 0;
	let navIndicatorWidth = 0;
	let navIndicatorTop = 0;
	let navIndicatorHeight = 0;
	let navButtons = [];

	$: {
		if (typeof window !== 'undefined' && innerHeight > 0) {
			const stepHeight = innerHeight;
			
			// Simple visibility: show content if we're at or past the section
			visibleSections.clear();
			for (let i = 0; i < steps.length; i++) {
				// Show content if we've scrolled at least 70% into the section
				if (scrollY >= (i * stepHeight) + (stepHeight * 0.3)) {
					visibleSections.add(i);
				}
			}
			
			const newCurrentStep = Math.min(Math.floor(scrollY / stepHeight), steps.length - 1);
			
			// Update URL when step changes due to scrolling (but not during programmatic navigation)
			if (newCurrentStep !== currentStep && !isNavigatingProgrammatically) {
				currentStep = newCurrentStep;
				updateURL(currentStep);
			}
			
			// Calculate scroll progress for the first section (0 to 1) - always based on actual scroll position
			scrollProgress = Math.min(scrollY / stepHeight, 1);
			
			// Track if we're transitioning from first page
			isTransitioning = scrollProgress > 0 && scrollProgress < 1;
			
			// Handle scroll hint
			hideScrollHint();
			
			// Interpolate header background height - responsive for mobile vs desktop
			const desktopHeaderHeight = 144; // 144px for desktop
			const mobileHeaderHeight = 16; // Much smaller for mobile (just top padding)
			headerHeight = scrollProgress * (typeof window !== 'undefined' && window.innerWidth < 640 ? mobileHeaderHeight : desktopHeaderHeight);
			
			// Calculate logo position - responsive for mobile vs desktop
			const isMobile = typeof window !== 'undefined' && window.innerWidth < 640;
			
			// Calculate proper center position based on actual viewport height
			// Center position = (viewport height - logo size) / 2
			const centerPosition = (innerHeight - logoSize) / 2;
			const finalPosition = 16; // Final top position in pixels
			
			if (isMobile) {
				// Mobile: animate towards left position
				const mobileLogoFinalLeft = 16; // 16px from left (matching mobile header padding)
				const centerX = (innerWidth - logoSize) / 2; // Center X position
				const finalX = mobileLogoFinalLeft;
				
				// Animate top position (same as before)
				const calculatedTopPosition = centerPosition - (scrollProgress * (centerPosition - 20)); // Final top position for mobile
				
				// Store both positions for mobile
				logoTopPosition = calculatedTopPosition;
				if (typeof window !== 'undefined') {
					window.mobileLogoLeftPosition = centerX - (scrollProgress * (centerX - finalX));
				}
			} else {
				// Desktop: original behavior
				const calculatedPosition = centerPosition - (scrollProgress * (centerPosition - finalPosition));
				
				// If the logo would go below the white background, expand the white background instead
				if (calculatedPosition + logoSize > headerHeight && headerHeight > 0) {
					headerHeight = Math.max(headerHeight, calculatedPosition + logoSize + 16); // +16px padding
				}
				
				logoTopPosition = calculatedPosition;
			}
			
			// Interpolate logo size from 384px to 80px
			logoSize = 384 - (scrollProgress * (384 - 80));
		}
	}

	function scrollToStep(stepIndex: number) {
		if (typeof window !== 'undefined') {
			// Set programmatic navigation flag to prevent URL interference
			isNavigatingProgrammatically = true;
			
			// Update current step immediately for instant indicator update
			currentStep = stepIndex;
			updateNavIndicator();
			
			window.scrollTo({
				top: stepIndex * innerHeight,
				behavior: 'smooth'
			});
			
			// Clear programmatic flag and finalize after scroll completes
			setTimeout(() => {
				isNavigatingProgrammatically = false;
				currentStep = stepIndex; // Ensure final step is set
				updateNavIndicator();
				// Don't call updateURL here - let the reactive scroll calculations handle it
			}, 1000);
		}
	}
	
	function updateNavIndicator() {
		if (currentStep > 0 && navButtons.length > 0) {
			const activeIndex = currentStep - 1;
			
			// Make sure we have the right button
			const button = navButtons[activeIndex];
			if (button && button.offsetWidth > 0) {
				const newLeft = button.offsetLeft;
				const newWidth = button.offsetWidth;
				const newTop = button.offsetTop;
				const newHeight = button.offsetHeight;
				navIndicatorLeft = newLeft;
				navIndicatorWidth = newWidth;
				navIndicatorTop = newTop;
				navIndicatorHeight = newHeight;
			}
		} else {
			// Hide indicator when on first step
			navIndicatorWidth = 0;
		}
	}
	
	// Update indicator when current step changes
	$: if (typeof window !== 'undefined' && currentStep !== undefined) {
		// Immediate update for responsiveness
		updateNavIndicator();
		// Also update with delay to ensure DOM is ready
		setTimeout(() => {
			updateNavIndicator();
		}, 50);
	}

	// Set Outfit font on mount
	$: {
		if (typeof document !== 'undefined') {
			document.documentElement.style.setProperty('--main-font', 'Outfit, sans-serif');
		}
	}

	onMount(() => {
		// Start scroll hint timer
		startScrollHintTimer();
		
		// Fix mobile viewport height
		const setVH = () => {
			const vh = window.innerHeight * 0.01;
			document.documentElement.style.setProperty('--vh', `${vh}px`);
		};
		
		setVH();
		window.addEventListener('resize', setVH);
		window.addEventListener('orientationchange', setVH);
		
		window.requestAnimationFrame(animate);
		
		// Initialize navigation indicator after a delay
		setTimeout(() => {
			updateNavIndicator();
		}, 500);
		
		// Scroll to initial section if specified
		if (initialSection > 0) {
			setTimeout(() => {
				isNavigatingProgrammatically = true;
				currentStep = initialSection;
				window.scrollTo({
					top: initialSection * innerHeight,
					behavior: 'instant' // Use instant for initial load
				});
				updateNavIndicator();
				
				// Allow normal operations after initial positioning
				setTimeout(() => {
					isNavigatingProgrammatically = false;
					isInitialLoad = false;
				}, 500);
			}, 100);
		} else {
			// Allow URL updates after initial load for home page
			setTimeout(() => {
				isInitialLoad = false;
			}, 500);
		}
		
		return () => {
			window.removeEventListener('resize', setVH);
			window.removeEventListener('orientationchange', setVH);
		};
	});

	let prevTimeStampLogo: number, prevTimeStampColor: number;

	function animate(timeStamp: number) {
		if (prevTimeStampColor === undefined) {
			prevTimeStampColor = timeStamp;
		}
		if (timeStamp >= prevTimeStampColor + 1200) {
			randomNum = Math.floor(Math.random() * 360);
			prevTimeStampColor = timeStamp
		} 
		if (prevTimeStampLogo === undefined) {
			prevTimeStampLogo = timeStamp;
		}
		if (timeStamp >= prevTimeStampLogo + 500) {
			if (changingLogo) {
				changingLogo.getRandomLogo();
			}
			prevTimeStampLogo = timeStamp
		}
		window.requestAnimationFrame(animate);
	}
</script>

<svelte:window bind:scrollY bind:innerHeight />

<!-- Scroll Hint Arrow -->
{#if showScrollHint && currentStep === 0}
<div class="fixed bottom-8 left-1/2 transform -translate-x-1/2 z-40 animate-bounce">
	<svg 
		width="24" 
		height="24" 
		viewBox="0 0 24 24" 
		fill="none" 
		class="animate-pulse"
	>
		<path 
			d="M7 10l5 5 5-5" 
			stroke="black" 
			stroke-width="2" 
			stroke-linecap="round" 
			stroke-linejoin="round"
		/>
	</svg>
</div>
{/if}

<!-- Expanding White Background - Desktop version -->
<div class="hidden sm:block fixed top-0 left-0 w-full z-30 bg-white" style="height: {headerHeight}px;"></div>

<!-- Mobile White Background - Just for logo transition -->
<div class="sm:hidden fixed top-0 left-0 w-full z-30 bg-white {currentStep > 0 && scrollProgress >= 1 ? 'hidden' : ''}" style="height: {Math.max(60, logoTopPosition + logoSize + 16)}px;"></div>

<!-- Fixed Header with Navigation - Only show navigation when scrolled -->
{#if currentStep > 0}
<header class="fixed top-0 left-0 w-full z-40 bg-white transition-all duration-700 ease-in-out">
	<!-- Desktop Layout - Center everything -->
	<div class="hidden sm:flex flex-col items-center py-4 pt-28">
		<!-- Navigation below logo -->
		<nav class="relative flex flex-wrap justify-center space-x-4 sm:space-x-8 px-4">
			<!-- Sliding background indicator -->
			<div 
				class="absolute bg-black rounded-lg transition-all duration-300 ease-in-out pointer-events-none"
				style="left: {navIndicatorLeft}px; width: {navIndicatorWidth}px; top: {navIndicatorTop}px; height: {navIndicatorHeight}px; opacity: {navIndicatorWidth > 0 ? 1 : 0};"
			></div>
			
			{#each steps.slice(1) as step, i}
				<button 
					bind:this={navButtons[i]}
					on:click={() => scrollToStep(i + 1)}
					class="relative z-10 transition-all duration-300 px-4 py-2 rounded-lg font-semibold
						{currentStep === i + 1 ? 'text-white' : 'text-slate-950 hover:text-black hover:underline'}"
				>
					{step.title}
				</button>
			{/each}
		</nav>
	</div>
	
	<!-- Mobile Layout - Logo left, Menu right -->
	<div class="sm:hidden flex items-center justify-between px-4 py-2">
		<!-- Mobile Logo -->
		<div class="w-12 h-12">
			<img src="/fuzue-logo.png" class="w-full h-full object-contain">
		</div>
		
		<!-- Mobile Navigation Menu -->
		<nav class="flex space-x-2">
			{#each steps.slice(1) as step, i}
				<button 
					on:click={() => scrollToStep(i + 1)}
					class="px-2 py-1 text-xs font-medium transition-all duration-300 rounded
						{currentStep === i + 1 ? 'bg-black text-white' : 'text-slate-950 hover:text-black'}"
				>
					{step.title}
				</button>
			{/each}
		</nav>
	</div>
</header>
{/if}

<!-- Transitioning Logo - Desktop only -->
<div class="hidden sm:flex fixed top-0 left-0 w-full h-full z-50 pointer-events-none justify-center">
	<div 
		style="width: {logoSize}px; height: {logoSize}px; transform: translateY({logoTopPosition}px);"
		class="transition-transform duration-75 ease-out"
	>
		<img src="/fuzue-logo.png" class="w-full h-full object-contain">
	</div>
</div>

<!-- Mobile Transitioning Logo - Animates to left position -->
<div class="sm:hidden fixed top-0 left-0 w-full h-full z-50 pointer-events-none {currentStep > 0 && scrollProgress >= 0.9 ? 'hidden' : ''}">
	<div 
		style="width: {logoSize}px; height: {logoSize}px; transform: translate({typeof window !== 'undefined' && window.mobileLogoLeftPosition ? window.mobileLogoLeftPosition : (typeof window !== 'undefined' ? (window.innerWidth - logoSize) / 2 : 0)}px, {logoTopPosition}px);"
		class="transition-transform duration-75 ease-out absolute"
	>
		<img src="/fuzue-logo.png" class="w-full h-full object-contain">
	</div>
</div>

<!-- Main Content Container -->
<div class="relative w-full">
	{#each steps as step, i}
		<section id={step.id} class="w-full relative" style:background-color={step.bgColor} style="height: calc(var(--vh, 1vh) * 100);">
			{#if i === 0}
				<!-- First Step: Just the logo -->
				<div class="w-full" style="height: calc(var(--vh, 1vh) * 100);">
					<!-- Logo is positioned by the fixed logo component -->
				</div>
			{:else}
				<!-- Other Steps: Content Slides -->
				<div class="w-full flex items-center justify-center" style="height: calc(var(--vh, 1vh) * 100);">
					<div class="w-full max-w-4xl px-6 sm:px-12">
						<!-- Animated Title -->
						<h1 class="w-full text-4xl sm:text-6xl font-bold {step.textColor} mb-8 transform transition-all duration-700 ease-out
							{scrollY >= i * innerHeight * 0.5 ? 'translate-x-0 opacity-100' : 'translate-x-full opacity-0'}"
						>
							{step.title}
						</h1>
						
						<!-- Services Section with Arrow Navigation -->
						{#if step.hasHorizontalScroll}
							<!-- Project with navigation - Full width, full height covering entire section -->
							<div class="absolute top-0 bottom-0 w-screen {step.projects[currentProjectIndex].bgColor} transition-all duration-500" 
								 style="left: 50%; right: 50%; margin-left: -50vw; margin-right: -50vw;">
								
								<!-- Desktop Left Arrow - Vertically centered, no background -->
								<button 
									on:click={prevProject}
									disabled={currentProjectIndex === 0}
									class="hidden sm:flex absolute left-2 top-1/2 -translate-y-1/2 w-12 h-12 transition-all duration-300 items-center justify-center disabled:opacity-30 disabled:cursor-not-allowed hover:scale-110"
								>
									<svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="black" stroke-width="3">
										<path d="M15 18l-6-6 6-6"/>
									</svg>
								</button>
								
								<!-- Mobile Left Arrow - Simple, no background -->
								<button 
									on:click={prevProject}
									disabled={currentProjectIndex === 0}
									class="sm:hidden absolute left-1 top-1/2 -translate-y-1/2 w-10 h-10 flex items-center justify-center transition-all duration-300 disabled:opacity-30 disabled:cursor-not-allowed"
								>
									<svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="black" stroke-width="3">
										<path d="M15 18l-6-6 6-6"/>
									</svg>
								</button>
								
								<!-- Current Project Content -->
								<div class="flex items-center justify-center h-full sm:px-20 px-8">
									<div class="max-w-4xl mx-auto text-center">
										<!-- Services title integrated into project area -->
										<h2 class="text-4xl sm:text-6xl font-bold text-gray-800 mb-8">{step.title}</h2>
										<h3 class="text-2xl sm:text-3xl md:text-4xl font-bold text-gray-800 mb-4 sm:mb-6">{step.projects[currentProjectIndex].title}</h3>
										<p class="text-base sm:text-lg md:text-xl text-gray-700 mb-4 sm:mb-6 leading-relaxed">{step.projects[currentProjectIndex].description}</p>
										<div class="text-sm sm:text-base text-gray-600 font-medium">
											<strong>Tech Stack:</strong> {step.projects[currentProjectIndex].tech}
										</div>
									</div>
								</div>
								
								<!-- Desktop Right Arrow - Vertically centered, no background -->
								<button 
									on:click={() => nextProject(step.projects.length)}
									disabled={currentProjectIndex === step.projects.length - 1}
									class="hidden sm:flex absolute right-2 top-1/2 -translate-y-1/2 w-12 h-12 transition-all duration-300 items-center justify-center disabled:opacity-30 disabled:cursor-not-allowed hover:scale-110"
								>
									<svg width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="black" stroke-width="3">
										<path d="M9 18l6-6-6-6"/>
									</svg>
								</button>
								
								<!-- Mobile Right Arrow - Simple, no background -->
								<button 
									on:click={() => nextProject(step.projects.length)}
									disabled={currentProjectIndex === step.projects.length - 1}
									class="sm:hidden absolute right-1 top-1/2 -translate-y-1/2 w-10 h-10 flex items-center justify-center transition-all duration-300 disabled:opacity-30 disabled:cursor-not-allowed"
								>
									<svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="black" stroke-width="3">
										<path d="M9 18l6-6-6-6"/>
									</svg>
								</button>
							</div>
						{:else}
							<!-- Regular content for other sections -->
							<div class="w-full text-lg sm:text-xl {step.bgColor === 'black' ? 'text-gray-300' : 'text-slate-700'} leading-relaxed transform transition-all duration-700 delay-200 ease-out
								{scrollY >= i * innerHeight * 0.5 ? 'translate-x-0 opacity-100' : 'translate-x-full opacity-0'}"
							>
								{step.content}
								
								{#if step.email}
									<div class="mt-8 text-2xl sm:text-3xl font-semibold {step.textColor}">
										{step.email}
									</div>
								{/if}
							</div>
						{/if}
					</div>
				</div>
			{/if}
		</section>
	{/each}
</div>