<script lang="ts">
    import {onMount, onDestroy} from 'svelte';

    // Color scheme options
    const COLOR_SCHEMES = {
        RAINBOW: 'rainbow',
        MUSIC_THEORY: 'music-theory',
        CHECKERBOARD: 'checkerboard',
        NEON_GLOW: 'neon-glow',
        MPC: 'mpc',
        MIDNIGHT: 'midnight' // New synthwave cyberpunk style
    };

    // Grid size options
    const GRID_SIZES = {
        GRID_3X3: '3x3',
        GRID_4X4: '4x4',
        GRID_5X5: '5x5',
        GRID_6X6: '6x6',
        GRID_7X7: '7x7',
        GRID_8X8: '8x8',
        GRID_12X2: '12x2'
    };

    // Grid dimensions mapping
    const GRID_DIMENSIONS = {
        [GRID_SIZES.GRID_3X3]: {rows: 3, cols: 3},
        [GRID_SIZES.GRID_4X4]: {rows: 4, cols: 4},
        [GRID_SIZES.GRID_5X5]: {rows: 5, cols: 5},
        [GRID_SIZES.GRID_6X6]: {rows: 6, cols: 6},
        [GRID_SIZES.GRID_7X7]: {rows: 7, cols: 7},
        [GRID_SIZES.GRID_8X8]: {rows: 8, cols: 8},
        [GRID_SIZES.GRID_12X2]: {rows: 2, cols: 12}
    };

    // State variables
    let midiAccess: MIDIAccess | null = null;
    let midiInputs: MIDIInput[] = [];
    let selectedInput: string = "";
    let pads: Pad[] = [];
    let error: string | null = null;
    let audioContext: AudioContext | null = null;
    let audioBuffers: Map<string, AudioBuffer> = new Map();
    let audioInitialized: boolean = false;
    let selectedColorScheme: ColorScheme = COLOR_SCHEMES.RAINBOW;
    let selectedGridSize: GridSize = GRID_SIZES.GRID_4X4;
    let gridTemplateColumns: string = "repeat(4, minmax(0, 1fr))";

    // Sound assignment variables
    let selectedPadForSound: Pad | null = null;
    let showSoundAssignModal: boolean = false;
    let customSoundNames: Map<number, string> = new Map(); // Maps pad ID to custom sound name

    // Initialize WebMIDI and load sounds
    onMount(() => {
        // Create audio context
        audioContext = new AudioContext();

        // Initialize MIDI
        if (navigator.requestMIDIAccess) {
            navigator.requestMIDIAccess()
                .then(onMIDISuccess)
                .catch(() => error = "Failed to access MIDI devices. Please check your browser permissions.");
        } else {
            error = "WebMIDI is not supported in your browser. Try using Chrome or Edge.";
        }

        // Initialize sound pads
        initializePads();

        // Load default sounds
        loadDefaultSounds(pads);

        // Add event listener to initialize audio on first interaction
        document.addEventListener('touchstart', initAudio, {once: true});
        document.addEventListener('mousedown', initAudio, {once: true});

        // Add global event listeners for mouseup/touchend
        document.addEventListener('mouseup', handleGlobalMouseUp);
        document.addEventListener('touchend', handleGlobalTouchEnd);
    });

    onDestroy(() => {
        if (audioContext) {
            audioContext.close();
        }

        // Remove global event listeners
        document.removeEventListener('mouseup', handleGlobalMouseUp);
        document.removeEventListener('touchend', handleGlobalTouchEnd);
    });

    // Update grid template columns based on selected grid size
    function updateGridLayout() {
        const {cols} = GRID_DIMENSIONS[selectedGridSize];
        gridTemplateColumns = `repeat(${cols}, minmax(0, 1fr))`;
    }

    // Initialize pads with the selected grid size and color scheme
    function initializePads() {
        const {rows, cols} = GRID_DIMENSIONS[selectedGridSize];
        const totalPads = rows * cols;
        const initialPads = [];

        // Starting MIDI note (C2 = 36)
        let startNote = 36;


        // For square grids, use the new note ordering:
        // Bottom left is lowest note, increasing to the right and then up
        for (let row = rows - 1; row >= 0; row--) {
            for (let col = 0; col < cols; col++) {
                const i = (rows - 1 - row) * cols + col;
                const note = startNote + i;
                initialPads.push({
                    id: i,
                    note: note,
                    soundFile: `default-sound-${i}`,
                    isActive: false,
                    isPressed: false,
                    row: row,
                    col: col,
                    noteName: getNoteName(note),
                    hasCustomSound: false
                });
            }
        }


        pads = initialPads;
        applyColorScheme();
        updateGridLayout();
    }

    // Get note name from MIDI note number
    function getNoteName(midiNote) {
        const noteNames = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];
        const noteName = noteNames[midiNote % 12];
        const octave = Math.floor(midiNote / 12) - 1;
        return `${noteName}${octave}`;
    }

    // Apply the selected color scheme to all pads
    function applyColorScheme() {
        pads = pads.map(pad => {
            const color = getPadColor(pad);
            return {...pad, color};
        });
    }

    // Get pad color based on the selected scheme
    function getPadColor(pad) {
        const {note, row, col, noteName} = pad;
        const noteValue = note % 12; // C=0, C#=1, etc.

        switch (selectedColorScheme) {
            case COLOR_SCHEMES.RAINBOW:
                // Original rainbow colors with more variety
                const hue = (noteValue * 30 + row * 15) % 360;
                return `hsl(${hue}, 85%, 65%)`;

            case COLOR_SCHEMES.MUSIC_THEORY:
                // Music theory colors - root, fourth, fifth
                if (noteValue === 0) {
                    return "#FF5252"; // Root note (C) - Red
                } else if (noteValue === 5) {
                    return "#448AFF"; // Fourth (F) - Blue
                } else if (noteValue === 7) {
                    return "#69F0AE"; // Fifth (G) - Green
                } else {
                    return "#9E9E9E"; // Other notes - Gray
                }

            case COLOR_SCHEMES.CHECKERBOARD:
                // Checkerboard pattern
                return (row + col) % 2 === 0 ? "#212121" : "#616161";

            case COLOR_SCHEMES.NEON_GLOW:
                // Neon glow - custom color scheme based on note value
                const neonHue = (noteValue * 30) % 360;
                return `hsl(${neonHue}, 100%, 65%)`;

            case COLOR_SCHEMES.MPC:
                // Blank MPC style - all pads same color
                return "#424242";

            case COLOR_SCHEMES.MIDNIGHT:
                // Midnight synthwave cyberpunk style
                // Use a combination of dark blues, purples, and pinks
                const synthwaveColors = [
                    "#0f1e45", // Dark blue
                    "#1a2151", // Navy blue
                    "#4a1e9e", // Deep purple
                    "#7b2fbe", // Bright purple
                    "#e92efb", // Hot pink
                    "#ff2a6d", // Neon pink
                    "#1eaedb", // Cyan
                    "#00fff5"  // Bright cyan
                ];

                // Create a gradient effect based on position and note value
                const colorIndex = (noteValue + row * 2) % synthwaveColors.length;
                return synthwaveColors[colorIndex];

            default:
                return "#616161";
        }
    }

    // Initialize audio on first user interaction
    const initAudio = () => {
        if (!audioInitialized && audioContext) {
            audioContext.resume().then(() => {
                audioInitialized = true;
            });
        }
    };

    // Handle MIDI access success
    const onMIDISuccess = (access) => {
        midiAccess = access;

        // Get list of inputs
        const inputs = [];
        midiAccess.inputs.forEach((input) => {
            inputs.push({
                id: input.id,
                name: input.name || `Unknown device (${input.id})`,
                manufacturer: input.manufacturer || 'Unknown manufacturer'
            });
        });

        midiInputs = inputs;

        // Listen for MIDI connection changes
        midiAccess.onstatechange = (e) => {
            if (e.port.type === 'input') {
                // Refresh the input list when devices change
                const updatedInputs = [];
                midiAccess.inputs.forEach((input) => {
                    updatedInputs.push({
                        id: input.id,
                        name: input.name || `Unknown device (${input.id})`,
                        manufacturer: input.manufacturer || 'Unknown manufacturer'
                    });
                });
                midiInputs = updatedInputs;
            }
        };
    };

    // Load default sounds (generated tones)
    const loadDefaultSounds = async (soundPads) => {
        if (!audioContext) return;

        try {
            // For this demo, we'll use generated sounds
            const ctx = audioContext;

            // Create a simple oscillator-based sound for each pad
            for (let i = 0; i < soundPads.length; i++) {
                const pad = soundPads[i];
                const frequency = 220 * Math.pow(2, (pad.note - 57) / 12); // A3 = 220Hz

                // Create a buffer for a simple sound
                const sampleRate = ctx.sampleRate;
                const buffer = ctx.createBuffer(2, sampleRate * 0.5, sampleRate);

                // Fill the buffer with a simple tone
                for (let channel = 0; channel < 2; channel++) {
                    const data = buffer.getChannelData(channel);
                    for (let j = 0; j < buffer.length; j++) {
                        // Simple sine wave with decay
                        data[j] = Math.sin(2 * Math.PI * frequency * j / sampleRate) *
                            Math.exp(-5 * j / buffer.length);
                    }
                }

                audioBuffers.set(pad.soundFile, buffer);
            }
        } catch (err) {
            console.error('Error loading sounds:', err);
            error = 'Failed to load sound files';
        }
    };

    // Load a custom WAV file
    const loadCustomSound = async (file, padId) => {
        if (!audioContext) return;

        try {
            // Read the file as an ArrayBuffer
            const arrayBuffer = await file.arrayBuffer();

            // Decode the audio data
            const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);

            // Find the pad
            const padIndex = pads.findIndex(pad => pad.id === padId);
            if (padIndex >= 0) {
                // Create a unique sound ID for this pad
                const soundId = `custom-sound-${padId}-${Date.now()}`;

                // Store the audio buffer
                audioBuffers.set(soundId, audioBuffer);

                // Update the pad with the new sound file
                pads[padIndex].soundFile = soundId;
                pads[padIndex].hasCustomSound = true;

                // Store the filename for display
                customSoundNames.set(padId, file.name);

                // Update the pads array to trigger reactivity
                pads = [...pads];
            }

            return true;
        } catch (err) {
            console.error('Error loading custom sound:', err);
            error = `Failed to load sound file: ${err.message}`;
            return false;
        }
    };

    // Handle MIDI input selection
    const handleInputChange = (event) => {
        const select = event.target;
        const value = select.value;

        if (!midiAccess) return;

        // Remove listeners from previous input
        if (selectedInput) {
            const prevInput = midiAccess.inputs.get(selectedInput);
            if (prevInput) {
                prevInput.onmidimessage = null;
            }
        }

        // Set up new input
        selectedInput = value;
        const input = midiAccess.inputs.get(value);

        if (input) {
            input.onmidimessage = handleMIDIMessage;
        }
    };

    // Handle incoming MIDI messages
    const handleMIDIMessage = (message) => {
        const data = message.data;

        // Check if it's a noteOn message (0x90)
        if ((data[0] & 0xf0) === 0x90) {
            const note = data[1];
            const velocity = data[2];

            // If velocity is 0, it's actually a note off message
            if (velocity > 0) {
                triggerPad(note);
            } else {
                releasePad(note);
            }
        }
        // Check if it's a noteOff message (0x80)
        else if ((data[0] & 0xf0) === 0x80) {
            const note = data[1];
            releasePad(note);
        }
    };

    // Trigger a pad when a MIDI note is received
    const triggerPad = (note) => {
        initAudio(); // Make sure audio is initialized

        const padIndex = pads.findIndex(pad => pad.note === note);

        if (padIndex >= 0) {
            // Update pad state to show it's active and pressed
            pads[padIndex].isActive = true;
            pads[padIndex].isPressed = true;
            pads = [...pads]; // Trigger reactivity

            // Play the sound
            playSound(pads[padIndex].soundFile);
        }
    };

    // Release a pad when a MIDI note off is received
    const releasePad = (note) => {
        const padIndex = pads.findIndex(pad => pad.note === note);

        if (padIndex >= 0) {
            // Keep isActive true for a moment for visual feedback
            pads[padIndex].isPressed = false;
            pads = [...pads]; // Trigger reactivity

            // After a short delay, turn off the active state
            setTimeout(() => {
                pads[padIndex].isActive = false;
                pads = [...pads]; // Trigger reactivity
            }, 100);
        }
    };

    // Play a sound from the loaded audio buffers
    const playSound = (soundFile) => {
        if (!audioContext) return;

        // Make sure audio context is running
        if (audioContext.state !== 'running') {
            audioContext.resume();
        }

        const buffer = audioBuffers.get(soundFile);
        if (buffer) {
            const source = audioContext.createBufferSource();
            source.buffer = buffer;
            source.connect(audioContext.destination);
            source.start(0); // Start immediately (0 seconds from now)
        }
    };

    // Handle pad mousedown
    const handlePadMouseDown = (pad) => {
        triggerPad(pad.note);
    };

    // Handle pad touchstart
    const handlePadTouchStart = (event, pad) => {
        event.preventDefault(); // Prevent default touch behavior
        triggerPad(pad.note);
    };

    // Handle pad right-click to assign sound
    const handlePadRightClick = (event, pad) => {
        event.preventDefault(); // Prevent context menu
        selectedPadForSound = pad;
        showSoundAssignModal = true;
    };

    // Handle pad long-press on mobile to assign sound
    let longPressTimer;
    const handlePadTouchStart_LongPress = (event, pad) => {
        longPressTimer = setTimeout(() => {
            selectedPadForSound = pad;
            showSoundAssignModal = true;
        }, 800); // 800ms long press
    };

    const handlePadTouchEnd_LongPress = () => {
        clearTimeout(longPressTimer);
    };

    // Handle global mouseup to release all pressed pads
    const handleGlobalMouseUp = () => {
        pads.forEach(pad => {
            if (pad.isPressed) {
                releasePad(pad.note);
            }
        });
    };

    // Handle global touchend to release all pressed pads
    const handleGlobalTouchEnd = () => {
        pads.forEach(pad => {
            if (pad.isPressed) {
                releasePad(pad.note);
            }
        });
    };

    // Handle color scheme change
    const handleColorSchemeChange = (event) => {
        selectedColorScheme = event.target.value;
        applyColorScheme();
    };

    // Handle grid size change
    const handleGridSizeChange = (event) => {
        selectedGridSize = event.target.value;
        initializePads();
        loadDefaultSounds(pads);
        customSoundNames = new Map(); // Reset custom sound names
    };

    // Handle file input change
    const handleFileInputChange = async (event) => {
        const file = event.target.files[0];
        if (!file) return;

        // Check if it's an audio file
        if (!file.type.startsWith('audio/')) {
            error = 'Please select an audio file (WAV, MP3, etc.)';
            return;
        }

        // Load the custom sound
        const success = await loadCustomSound(file, selectedPadForSound.id);

        if (success) {
            showSoundAssignModal = false;
        }

        // Reset the file input
        event.target.value = '';
    };

    // Get glow color based on pad color
    function getGlowColor(color) {
        // For named colors like #424242, convert to RGB
        if (color.startsWith('#')) {
            // For Midnight theme, make the glow more intense
            if (selectedColorScheme === COLOR_SCHEMES.MIDNIGHT) {
                // Extract the hex color and make it brighter for the glow
                const r = parseInt(color.slice(1, 3), 16);
                const g = parseInt(color.slice(3, 5), 16);
                const b = parseInt(color.slice(5, 7), 16);

                // Brighten the color for the glow
                const brightenedColor = `rgba(${Math.min(r + 50, 255)}, ${Math.min(g + 50, 255)}, ${Math.min(b + 100, 255)}, 0.8)`;
                return brightenedColor;
            }
            return color;
        }
        // For HSL colors, extract the hue and create a brighter version
        if (color.startsWith('hsl')) {
            const hueMatch = color.match(/hsl\((\d+)/);
            if (hueMatch && hueMatch[1]) {
                const hue = hueMatch[1];
                return `hsl(${hue}, 100%, 70%)`;
            }
        }
        return color;
    }

    // Close the sound assignment modal
    const closeModal = () => {
        showSoundAssignModal = false;
        selectedPadForSound = null;
    };

    // Get additional styles for Midnight theme
    function getMidnightStyles() {
        if (selectedColorScheme !== COLOR_SCHEMES.MIDNIGHT) return '';

        // Add a grid overlay for the cyberpunk effect
        return `
      background-image: linear-gradient(0deg, rgba(0,0,0,0.3) 1px, transparent 1px),
                        linear-gradient(90deg, rgba(0,0,0,0.3) 1px, transparent 1px);
      background-size: 5px 5px;
      background-position: center center;
    `;
    }
</script>

<div class="flex min-h-screen flex-col items-center justify-center p-4 bg-gray-900">
    <h1 class="text-3xl font-bold text-white mb-6">A Sound Bored</h1>

    <div class="flex flex-col items-center w-full max-w-4xl gap-6">
        {#if error}
            <div class="w-full p-4 mb-4 bg-red-500 text-white rounded-md">
                {error}
            </div>
        {/if}

        <div class="w-full grid grid-cols-1 md:grid-cols-3 gap-4">
            <div>
                <h2 class="text-xl font-semibold text-white mb-2">MIDI Input</h2>
                <select
                        class="w-full p-2 rounded-md bg-gray-800 text-white border border-gray-700"
                        value={selectedInput}
                        on:change={handleInputChange}
                >
                    <option value="" disabled selected>Select MIDI device</option>
                    {#if midiInputs.length === 0}
                        <option value="none" disabled>No MIDI devices found</option>
                    {:else}
                        {#each midiInputs as input}
                            <option value={input.id}>
                                {input.name} ({input.manufacturer})
                            </option>
                        {/each}
                    {/if}
                </select>
            </div>

            <div>
                <h2 class="text-xl font-semibold text-white mb-2">Color Scheme</h2>
                <select
                        class="w-full p-2 rounded-md bg-gray-800 text-white border border-gray-700"
                        value={selectedColorScheme}
                        on:change={handleColorSchemeChange}
                >
                    <option value={COLOR_SCHEMES.RAINBOW}>Original Rainbow</option>
                    <option value={COLOR_SCHEMES.MUSIC_THEORY}>Music Theory (Root, 4th, 5th)</option>
                    <option value={COLOR_SCHEMES.CHECKERBOARD}>Checkerboard</option>
                    <option value={COLOR_SCHEMES.NEON_GLOW}>Neon Glow</option>
                    <option value={COLOR_SCHEMES.MPC}>Blank MPC Style</option>
                    <option value={COLOR_SCHEMES.MIDNIGHT}>Midnight Synthwave</option>
                </select>
            </div>

            <div>
                <h2 class="text-xl font-semibold text-white mb-2">Grid Size</h2>
                <select
                        class="w-full p-2 rounded-md bg-gray-800 text-white border border-gray-700"
                        value={selectedGridSize}
                        on:change={handleGridSizeChange}
                >
                    <option value={GRID_SIZES.GRID_3X3}>3×3 (9 pads)</option>
                    <option value={GRID_SIZES.GRID_4X4}>4×4 (16 pads)</option>
                    <option value={GRID_SIZES.GRID_5X5}>5×5 (25 pads)</option>
                    <option value={GRID_SIZES.GRID_6X6}>6×6 (36 pads)</option>
                    <option value={GRID_SIZES.GRID_7X7}>7×7 (49 pads)</option>
                    <option value={GRID_SIZES.GRID_8X8}>8×8 (64 pads)</option>
                    <option value={GRID_SIZES.GRID_12X2}>12×2 (24 pads, 2 octaves)</option>
                </select>
            </div>
        </div>

        <div class="w-full">
            <div class="flex justify-between items-center mb-4">
                <h2 class="text-xl font-semibold text-white">Sound Pads</h2>
                <div class="text-sm text-gray-400">
                    <span>Right-click or long-press a pad to assign a sound</span>
                </div>
            </div>

            <div class="grid gap-2" style="grid-template-columns: {gridTemplateColumns};">
                {#each pads as pad}
                    <button
                            class="aspect-square p-0 rounded-md relative overflow-hidden pad-button"
                            style="
              order: {pad.row - 12};
              background-color: {pad.color};
              transform: {pad.isPressed ? 'translateY(2px)' : 'translateY(0)'};
              transition: transform 0.05s ease, box-shadow 0.05s ease, filter 0.1s ease;
              box-shadow: {pad.isPressed || pad.isActive
                ? `inset 0 2px 5px rgba(0,0,0,0.5), 0 0 ${pad.isActive ? '15px 5px' : '0'} ${getGlowColor(pad.color)}`
                : 'inset 0 1px 1px rgba(255,255,255,0.2), 0 3px 5px rgba(0,0,0,0.3)'};
              filter: {pad.isActive ? 'brightness(1.4)' : 'brightness(1)'};
              ${getMidnightStyles()}
            "
                            on:mousedown={() => handlePadMouseDown(pad)}
                            on:touchstart={(e) => handlePadTouchStart(e, pad)}
                            on:contextmenu={(e) => handlePadRightClick(e, pad)}
                            on:touchstart={(e) => {
              handlePadTouchStart(e, pad);
              handlePadTouchStart_LongPress(e, pad);
            }}
                            on:touchend={handlePadTouchEnd_LongPress}
                    >
                        <span class="text-xs text-white/70 absolute bottom-1 right-1">{pad.noteName}</span>
                        {#if selectedColorScheme === COLOR_SCHEMES.MUSIC_THEORY && (pad.note % 12 === 0 || pad.note % 12 === 5 || pad.note % 12 === 7)}
              <span class="absolute top-1 left-1 text-xs text-white/70">
                {pad.note % 12 === 0 ? 'R' : pad.note % 12 === 5 ? '4th' : '5th'}
              </span>
                        {/if}

                        {#if pad.hasCustomSound}
                            <div class="absolute inset-0 flex items-center justify-center">
                <span class="text-xs text-white/90 bg-black/30 px-1 py-0.5 rounded truncate max-w-[90%]">
                  {customSoundNames.get(pad.id) || 'Custom'}
                </span>
                            </div>
                        {/if}

                        {#if selectedColorScheme === COLOR_SCHEMES.MIDNIGHT}
                            <div class="absolute inset-0 midnight-overlay"></div>
                        {/if}
                    </button>
                {/each}
            </div>
        </div>

        <div class="text-sm text-gray-400 mt-4">
            <p>Connect a MIDI controller and press keys to trigger sounds.</p>
            <p>You can also tap/click on the pads directly for immediate sound.</p>
        </div>
    </div>

    {#if showSoundAssignModal}
        <div class="fixed inset-0 bg-black/70 flex items-center justify-center z-50">
            <div class="bg-gray-800 p-6 rounded-lg max-w-md w-full">
                <h3 class="text-xl font-semibold text-white mb-4">
                    Assign Sound to Pad {selectedPadForSound.noteName}
                </h3>

                <div class="mb-4">
                    <label class="block text-white mb-2" for="sound_assign">Select WAV or audio file:</label>
                    <input id="sound_assign"
                           type="file"
                           accept="audio/*"
                           class="w-full p-2 rounded-md bg-gray-700 text-white"
                           on:change={handleFileInputChange}
                    />
                </div>

                <div class="flex justify-end gap-2">
                    <button
                            class="px-4 py-2 bg-gray-700 text-white rounded-md hover:bg-gray-600"
                            on:click={closeModal}
                    >
                        Cancel
                    </button>
                </div>
            </div>
        </div>
    {/if}
</div>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
        font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
        Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
        overscroll-behavior: none; /* Prevent pull-to-refresh on mobile */
        touch-action: manipulation; /* Disable double-tap to zoom */
        background-color: #121212;
    }

    /* Add some depth to the buttons */
    .pad-button {
        border: 1px solid rgba(0, 0, 0, 0.2);
        user-select: none;
        -webkit-tap-highlight-color: transparent;
        position: relative;
    }

    /* Add a subtle glow effect that only appears on interaction */
    .pad-button::after {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        border-radius: inherit;
        pointer-events: none;
        background: radial-gradient(circle at center, rgba(255, 255, 255, 0.3) 0%, rgba(255, 255, 255, 0) 70%);
        opacity: 0;
        transition: opacity 0.2s ease;
    }

    .pad-button:active::after {
        opacity: 1;
    }

    /* Midnight theme specific styles */
    .midnight-overlay {
        pointer-events: none;
        background: linear-gradient(135deg, rgba(0, 0, 0, 0) 0%, rgba(0, 0, 0, 0.2) 100%);
        border-top: 1px solid rgba(255, 255, 255, 0.1);
        border-left: 1px solid rgba(255, 255, 255, 0.1);
    }

    /* Responsive adjustments for different grid sizes */
    @media (max-width: 768px) {
        .grid {
            gap: 4px;
        }
    }
</style>

