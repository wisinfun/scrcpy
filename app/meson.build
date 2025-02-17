src = [
    'src/main.c',
    'src/adb/adb.c',
    'src/adb/adb_device.c',
    'src/adb/adb_parser.c',
    'src/adb/adb_tunnel.c',
    'src/cli.c',
    'src/clock.c',
    'src/compat.c',
    'src/control_msg.c',
    'src/controller.c',
    'src/decoder.c',
    'src/demuxer.c',
    'src/device_msg.c',
    'src/icon.c',
    'src/file_pusher.c',
    'src/fps_counter.c',
    'src/frame_buffer.c',
    'src/input_manager.c',
    'src/keyboard_inject.c',
    'src/mouse_inject.c',
    'src/opengl.c',
    'src/options.c',
    'src/receiver.c',
    'src/recorder.c',
    'src/scrcpy.c',
    'src/screen.c',
    'src/server.c',
    'src/version.c',
    'src/video_buffer.c',
    'src/util/acksync.c',
    'src/util/file.c',
    'src/util/intmap.c',
    'src/util/intr.c',
    'src/util/log.c',
    'src/util/net.c',
    'src/util/net_intr.c',
    'src/util/process.c',
    'src/util/process_intr.c',
    'src/util/strbuf.c',
    'src/util/str.c',
    'src/util/term.c',
    'src/util/thread.c',
    'src/util/tick.c',
]

conf = configuration_data()

conf.set('_POSIX_C_SOURCE', '200809L')
conf.set('_XOPEN_SOURCE', '700')
conf.set('_GNU_SOURCE', true)

if host_machine.system() == 'windows'
    windows = import('windows')
    src += [
        'src/sys/win/file.c',
        'src/sys/win/process.c',
        windows.compile_resources('scrcpy-windows.rc'),
    ]
    conf.set('_WIN32_WINNT', '0x0600')
    conf.set('WINVER', '0x0600')
else
    src += [
        'src/sys/unix/file.c',
        'src/sys/unix/process.c',
    ]
    if host_machine.system() == 'darwin'
        conf.set('_DARWIN_C_SOURCE', true)
    endif
endif

v4l2_support = get_option('v4l2') and host_machine.system() == 'linux'
if v4l2_support
    src += [ 'src/v4l2_sink.c' ]
endif

usb_support = get_option('usb')
if usb_support
    src += [
        'src/usb/aoa_hid.c',
        'src/usb/hid_keyboard.c',
        'src/usb/hid_mouse.c',
        'src/usb/scrcpy_otg.c',
        'src/usb/screen_otg.c',
        'src/usb/usb.c',
    ]
endif

cc = meson.get_compiler('c')

crossbuild_windows = meson.is_cross_build() and host_machine.system() == 'windows'

if not crossbuild_windows

    # native build
    dependencies = [
        dependency('libavformat', version: '>= 57.33'),
        dependency('libavcodec', version: '>= 57.37'),
        dependency('libavutil'),
        dependency('sdl2', version: '>= 2.0.5'),
    ]

    if v4l2_support
        dependencies += dependency('libavdevice')
    endif

    if usb_support
        dependencies += dependency('libusb-1.0')
    endif

else
    # cross-compile mingw32 build (from Linux to Windows)
    prebuilt_sdl2 = meson.get_cross_property('prebuilt_sdl2')
    sdl2_bin_dir = meson.current_source_dir() + '/prebuilt-deps/data/' + prebuilt_sdl2 + '/bin'
    sdl2_lib_dir = meson.current_source_dir() + '/prebuilt-deps/data/' + prebuilt_sdl2 + '/lib'
    sdl2_include_dir = 'prebuilt-deps/data/' + prebuilt_sdl2 + '/include'

    sdl2 = declare_dependency(
        dependencies: [
            cc.find_library('SDL2', dirs: sdl2_bin_dir),
            cc.find_library('SDL2main', dirs: sdl2_lib_dir),
        ],
        include_directories: include_directories(sdl2_include_dir)
    )

    prebuilt_ffmpeg = meson.get_cross_property('prebuilt_ffmpeg')
    ffmpeg_bin_dir = meson.current_source_dir() + '/prebuilt-deps/data/' + prebuilt_ffmpeg + '/bin'
    ffmpeg_include_dir = 'prebuilt-deps/data/' + prebuilt_ffmpeg + '/include'

    # ffmpeg versions are different for win32 and win64 builds
    ffmpeg_avcodec = meson.get_cross_property('ffmpeg_avcodec')
    ffmpeg_avformat = meson.get_cross_property('ffmpeg_avformat')
    ffmpeg_avutil = meson.get_cross_property('ffmpeg_avutil')

    ffmpeg = declare_dependency(
        dependencies: [
            cc.find_library(ffmpeg_avcodec, dirs: ffmpeg_bin_dir),
            cc.find_library(ffmpeg_avformat, dirs: ffmpeg_bin_dir),
            cc.find_library(ffmpeg_avutil, dirs: ffmpeg_bin_dir),
        ],
        include_directories: include_directories(ffmpeg_include_dir)
    )

    prebuilt_libusb = meson.get_cross_property('prebuilt_libusb')
    prebuilt_libusb_root = meson.get_cross_property('prebuilt_libusb_root')
    libusb_bin_dir = meson.current_source_dir() + '/prebuilt-deps/data/' + prebuilt_libusb + '/dll'
    libusb_include_dir = 'prebuilt-deps/data/' + prebuilt_libusb_root + '/include'

    libusb = declare_dependency(
        dependencies: [
            cc.find_library('libusb-1.0', dirs: libusb_bin_dir),
        ],
        include_directories: include_directories(libusb_include_dir)
    )

    dependencies = [
        ffmpeg,
        sdl2,
        libusb,
        cc.find_library('mingw32')
    ]

endif

if host_machine.system() == 'windows'
    dependencies += cc.find_library('ws2_32')
endif

check_functions = [
    'strdup',
    'asprintf',
    'vasprintf',
]

foreach f : check_functions
  if cc.has_function(f)
    define = 'HAVE_' + f.underscorify().to_upper()
    conf.set(define, true)
  endif
endforeach

conf.set('HAVE_SOCK_CLOEXEC', host_machine.system() != 'windows' and
                              cc.has_header_symbol('sys/socket.h', 'SOCK_CLOEXEC'))

# the version, updated on release
conf.set_quoted('SCRCPY_VERSION', meson.project_version())

# the prefix used during configuration (meson --prefix=PREFIX)
conf.set_quoted('PREFIX', get_option('prefix'))

# build a "portable" version (with scrcpy-server accessible from the same
# directory as the executable)
conf.set('PORTABLE', get_option('portable'))

# the default client TCP port range for the "adb reverse" tunnel
# overridden by option --port
conf.set('DEFAULT_LOCAL_PORT_RANGE_FIRST', '27183')
conf.set('DEFAULT_LOCAL_PORT_RANGE_LAST', '27199')

# the default video bitrate, in bits/second
# overridden by option --bit-rate
conf.set('DEFAULT_BIT_RATE', '8000000')  # 8Mbps

# run a server debugger and wait for a client to be attached
conf.set('SERVER_DEBUGGER', get_option('server_debugger'))

# select the debugger method ('old' for Android < 9, 'new' for Android >= 9)
conf.set('SERVER_DEBUGGER_METHOD_NEW', get_option('server_debugger_method') == 'new')

# enable V4L2 support (linux only)
conf.set('HAVE_V4L2', v4l2_support)

# enable HID over AOA support (linux only)
conf.set('HAVE_USB', usb_support)

configure_file(configuration: conf, output: 'config.h')

src_dir = include_directories('src')

executable('scrcpy', src,
           dependencies: dependencies,
           include_directories: src_dir,
           install: true,
           c_args: [])

install_man('scrcpy.1')
install_data('data/icon.png',
             rename: 'scrcpy.png',
             install_dir: 'share/icons/hicolor/256x256/apps')
install_data('data/zsh-completion/_scrcpy',
             install_dir: 'share/zsh/site-functions')
install_data('data/bash-completion/scrcpy',
             install_dir: 'share/bash-completion/completions')


### TESTS

# do not build tests in release (assertions would not be executed at all)
if get_option('buildtype') == 'debug'
    tests = [
        ['test_adb_parser', [
            'tests/test_adb_parser.c',
            'src/adb/adb_device.c',
            'src/adb/adb_parser.c',
            'src/util/str.c',
            'src/util/strbuf.c',
        ]],
        ['test_buffer_util', [
            'tests/test_buffer_util.c',
        ]],
        ['test_cbuf', [
            'tests/test_cbuf.c',
        ]],
        ['test_cli', [
            'tests/test_cli.c',
            'src/cli.c',
            'src/options.c',
            'src/util/net.c',
            'src/util/str.c',
            'src/util/strbuf.c',
            'src/util/term.c',
        ]],
        ['test_clock', [
            'tests/test_clock.c',
            'src/clock.c',
        ]],
        ['test_control_msg_serialize', [
            'tests/test_control_msg_serialize.c',
            'src/control_msg.c',
            'src/util/str.c',
            'src/util/strbuf.c',
        ]],
        ['test_device_msg_deserialize', [
            'tests/test_device_msg_deserialize.c',
            'src/device_msg.c',
        ]],
        ['test_queue', [
            'tests/test_queue.c',
        ]],
        ['test_strbuf', [
            'tests/test_strbuf.c',
            'src/util/strbuf.c',
        ]],
        ['test_str', [
            'tests/test_str.c',
            'src/util/str.c',
            'src/util/strbuf.c',
        ]],
        ['test_vector', [
            'tests/test_vector.c',
        ]],
    ]

    foreach t : tests
        exe = executable(t[0], t[1],
                         include_directories: src_dir,
                         dependencies: dependencies,
                         c_args: ['-DSDL_MAIN_HANDLED', '-DSC_TEST'])
        test(t[0], exe)
    endforeach
endif
