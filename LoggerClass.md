## Logger class to use with AsyncLogger with Colors

### Header (Logger.hpp)

    #pragma once
    #include <AsyncLogger/Logger.hpp>
    
    using namespace al;

    namespace rar
    {
    #define ADD_COLOR_TO_STREAM(color) "\x1b[" << int(color) << "m"
    #define RESET_STREAM_COLOR "\x1b[" << int(LogColor::RESET) << "m"
    #define HEX_TO_UPPER(value)                                        \
    "0x" << std::hex << std::uppercase << (DWORD64)value << std::dec \
     << std::nouppercase

	    enum class LogColor
	    {
		    RESET,
		    WHITE = 97,
		    CYAN = 36,
		    MAGENTA = 35,
		    BLUE = 34,
		    GREEN = 32,
		    YELLOW = 33,
		    RED = 31,
		    BRIGHT_RED = 91,
		    BLACK = 30
	    };

	    class logger final
	    {
	    public:
	    	logger();
	    	virtual ~logger();

	    	void initialize();
	    	void destroy();

	    private:
	    	void open_outstreams();
	    	void close_outstreams();

	    	void format_console(const LogMessagePtr msg);
	    	void format_console_simple(const LogMessagePtr msg);
	    	void format_file(const LogMessagePtr msg);

	    	void (logger::*m_console_logger)(const LogMessagePtr msg);

	    	std::ofstream m_console_out_;
	    	std::ofstream m_file_out_;
	    };

	        inline logger* g_log = nullptr;
    } // namespace rar

### Implemetation (Logger.cpp)

    #include "logger.hpp"
    
    #include <iostream>
    
    namespace rar
    {
    	template <typename Tp>
    	std::time_t ToTimeT(Tp tp)
    	{
    		using namespace std::chrono;
    		auto sctp = time_point_cast<system_clock::duration>(tp - Tp::clock::now() +
    			system_clock::now());
    		return system_clock::to_time_t(sctp);
    	}
    
    	logger::logger() : m_console_logger(&logger::format_console)
    	{
    		initialize();
    		g_log = this;
    	}
    
    	logger::~logger() { g_log = nullptr; }
    
    	void logger::initialize()
    	{
    		open_outstreams();
    
    		Logger::Init();
    		Logger::AddSink(
    			[this](LogMessagePtr msg) { (this->*m_console_logger)(std::move(msg)); });
    		Logger::AddSink([this](LogMessagePtr msg) { format_file(std::move(msg)); });
    	}
    
    	void logger::destroy()
    	{
    		Logger::Destroy();
    		close_outstreams();
    	}
    
    	void logger::open_outstreams()
    	{
    #ifdef _WIN32
    		const auto consoleHandle = GetStdHandle(STD_OUTPUT_HANDLE);
    
    		DWORD consoleMode;
    		GetConsoleMode(consoleHandle, &consoleMode);
    
    		consoleMode |=
    			ENABLE_VIRTUAL_TERMINAL_PROCESSING | DISABLE_NEWLINE_AUTO_RETURN;
    		SetConsoleMode(consoleHandle, consoleMode);
    #endif
    
    		m_console_out_.open("CONOUT$", std::ios::out | std::ios::app);
    		m_file_out_.open("./cout.log", std::ios::out | std::ios::trunc);
    	}
    
    	void logger::close_outstreams()
    	{
    		m_console_out_.close();
    		m_file_out_.close();
    	}
    
    	const LogColor GetColor(const eLogLevel level)
    	{
    		switch (level)
    		{
    		case VERBOSE:
    			return LogColor::BLUE;
    		case SUCCESS:
    			return LogColor::GREEN;
    		case INFO:
    			return LogColor::WHITE;
    		case WARNING:
    			return LogColor::YELLOW;
    		case ERROR1:
    			return LogColor::BRIGHT_RED;
    		case FATAL:
    			return LogColor::RED;
    		}
    		return LogColor::WHITE;
    	}
    
    	const char* get_level_string(const eLogLevel level)
    	{
    		constexpr std::array<const char*, 6> LevelStrings = {
    			{{"DEBUG"}, {"SUCCESS"}, {"INFO"}, {"WARN"}, {"ERROR"}, {"FATAL"}}
    		};
    		return LevelStrings[level];
    	}
    
    	void rar::logger::format_console(const LogMessagePtr msg)
    	{
    		const auto color = GetColor(msg->Level());
    
    		const auto timestamp = std::format("{0:%H:%M:%S}", msg->Timestamp());
    		const auto& location = msg->Location();
    		const auto level = msg->Level();
    		const auto file =
    			std::filesystem::path(location.file_name()).filename().string();
    
    		m_console_out_ << "[" << timestamp << "]" << ADD_COLOR_TO_STREAM(color) << "["
    			<< get_level_string(level) << "/" << file << ":" << location.line()
    			<< "] " << RESET_STREAM_COLOR << msg->Message() << std::flush;
    	}
    
    	void rar::logger::format_console_simple(const LogMessagePtr msg)
    	{
    		const auto color = GetColor(msg->Level());
    
    		const auto timestamp = std::format("{0:%H:%M:%S}", msg->Timestamp());
    		const auto& location = msg->Location();
    		const auto level = msg->Level();
    
    		const auto file =
    			std::filesystem::path(location.file_name()).filename().string();
    
    		m_console_out_ << "[" << timestamp << "]"
    			<< "[" << get_level_string(level) << "/" << file << ":"
    			<< location.line() << "] " << msg->Message() << std::flush;
    	}
    
    	void rar::logger::format_file(const LogMessagePtr msg)
    	{
    		const auto timestamp = std::format("{0:%H:%M:%S}", msg->Timestamp());
    		const auto& location = msg->Location();
    		const auto level = msg->Level();
    
    		const auto file =
    			std::filesystem::path(location.file_name()).filename().string();
    
    		m_file_out_ << "[" << timestamp << "]"
    			<< "[" << get_level_string(level) << "/" << file << ":"
    			<< location.line() << "] " << msg->Message() << std::flush;
    	}
    } // namespace rar

