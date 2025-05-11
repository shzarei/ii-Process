# Modifications

This document shows the different modifications that were introduced to simplify the use of code or to add a useful feature. These modification can be activated by adding the following definition before include the header file:

```cpp
#define EXTEND_ADAT
#include <adat/cxxopts.hpp>
```

Or add the definition in the project. in cmake project it can be done by adding ```add_definitions(-DEXTEND_ADAT)```.

 
# cxxopts:

In cxxopts the following section was added to create ```check_for_required_options``` function in order to verify the presence of required argument:

```c++
// line 1252
#ifdef EXTEND_ADAT
    bool
    check_for_required_options(std::vector<std::string> required_options) const;
#endif

// line 1847
#ifdef EXTEND_ADAT
inline
bool
ParseResult::check_for_required_options(std::vector<std::string> required_options) const
{

    for(auto& required_option: required_options){
        auto iter = m_options->find(required_option);
        if (iter != m_options->end())
        {
            if(!this->count(required_option)){
                throw_or_mimic<option_requires_argument_exception>(required_option);
                return false;
            }
        }
    }

    return true;
}
#endif
```

Example of how to use this function:

```c++
void parse(int argc, const char** argv)
{
    cxxopts::Options options("simple", "A brief description");

    options.add_options()
        ("b,bar", "Param bar (required)", cxxopts::value<std::string>())
        ("d,debug", "Enable debugging", cxxopts::value<bool>()->default_value("false"))
        ("f,foo", "Param foo", cxxopts::value<int>()->default_value("10"))
        ("h,help", "Print usage")
    ;

    auto result = options.parse(argc, argv);
    result.check_for_required_options({"bar"});


    if (result.count("help"))
    {
      std::cout << options.help() << std::endl;
      exit(0);
    }
    bool debug = result["debug"].as<bool>();

    std::string bar;
    if (result.count("bar"))
      bar = result["bar"].as<std::string>();

    if(debug)
        std::cout << "Debug mode activated!\n";

    int foo = result["foo"].as<int>();
    std::cout << fmt::format("foo = {}\n", foo);

}

```


## Indicators

Here we added a child class in order to ease Indicators integration in for loops with fewer lines of code.

```c++
class Iteration_progress_bar : public indicators::ProgressBar {
public:
    Iteration_progress_bar (const size_t size, const std::string prefix = ""):
        m_size(size)
    {
        std::string pre = prefix == ""? "Progress :": prefix;
        namespace iopt = indicators::option;
        this->set_option(iopt::BarWidth{50});
        this->set_option(iopt::ShowElapsedTime{true});
        this->set_option(iopt::ShowRemainingTime{true});
        this->set_option(iopt::PostfixText{"0/" + std::to_string(size)});
        this->set_option(iopt::PrefixText{pre});
        this->set_option(iopt::ForegroundColor{indicators::Color::green});
        this->set_option(iopt::FontStyles{std::vector<indicators::FontStyle>{indicators::FontStyle::bold}});
        this->set_option(iopt::ShowPercentage{true});
    }

    void tick(){
        m_iter++;
        size_t internal_progress = static_cast<size_t>(m_iter * 100 / m_size);
        if(internal_progress == m_progress)
            return;
        if(internal_progress == 100 && m_iter <= m_size - 1)
            return;
        m_progress = internal_progress;
        this->set_option(indicators::option::PostfixText{
                             std::to_string(m_iter) + "/" + std::to_string(m_size)});
        this->set_progress(m_progress);


    }
private:
    size_t m_size {0};
    size_t m_iter {0};
    size_t m_progress {0};

};
```

Example of how to use this class:

```c++
int main(int argc, const char** argv)
{
    size_t size = 15;
    Iteration_progress_bar bar(size);
    for(uint i = 0; i < size; i++){
        bar.tick();
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }

    // Update bar state
    return 0;
}

```

# tabluate

For tabulate library we added a child class for a default tabulate with fewer lines of code.

```c++
class Default_tabulate {
public:
    Default_tabulate(const uint num_columns):
        m_num_columns(num_columns)
    {

    }
    void set_first_row_as_title(const bool first_row_as_title)
    {
        m_first_row_as_title = first_row_as_title;
    }
    void set_add_padding_flag(const bool add_padding)
    {
        m_add_padding = add_padding;
    }
    void add_row(const std::vector<std::string>& row)
    {
        std::vector<variant<std::string, const char *, tabulate::Table>> line;
        for(uint i = 0; i < row.size(); i++){
            line.push_back(static_cast<variant<std::string, const char *, tabulate::Table>>(row.at(i)));
            if(i == (m_num_columns - 1))
                break;
        }
        m_table.add_row(line);
        m_num_rows++;
        if(!m_first_row_as_title || m_num_rows < 3)
            return;
        if(m_add_padding)
            m_table.row(m_num_rows - 1).format().padding_top(1);
        m_table.row(m_num_rows - 1).format().hide_border_top();

    }

    std::string str()
    {
        if(m_num_rows == 0)
            return "";

        m_table.row(0).format().font_align(tabulate::FontAlign::center);
        return m_table.str();
    }
private:
    tabulate::Table m_table;
    uint m_num_columns {0};
    uint m_num_rows {0};
    bool m_first_row_as_title {true};
    bool m_add_padding {false};
};
```

Example of how to use this class:

```c++
    Default_tabulate table(3);
    table.add_row({"Company", "Contact", "Country"});
    table.add_row({"Alfreds Futterkiste", "Maria Anders", "Germany"});
    table.add_row({"Centro comercial Moctezuma" , "Francisco Chang", "Mexico", "mistake cell"});
    table.add_row({"Ernst Handel", "Roland Mendel", "Austria"});
    table.add_row({"Island Trading", "Helen Bennett", "UK"});
    table.add_row({"Laughing Bacchus Winecellars", "Yoshi Tannamuri"});
    table.add_row({"Magazzini Alimentari Riuniti", "Giovanni Rovelli", "Italy"});

    std::cout << table.str() << std::endl;
```

Example of how without this class:

```c++
    tabulate::Table table2;
    table2.add_row({"Company", "Contact", "Country"});
    table2.add_row({"Alfreds Futterkiste", "Maria Anders", "Germany"});
    table2.add_row({"Centro comercial Moctezuma" , "Francisco Chang", "Mexico"});
    table2.add_row({"Ernst Handel", "Roland Mendel", "Austria"});
    table2.add_row({"Island Trading", "Helen Bennett", "UK"});
    table2.add_row({"Laughing Bacchus Winecellars", "Yoshi Tannamuri", "spain"});
    table2.add_row({"Magazzini Alimentari Riuniti", "Giovanni Rovelli", "Italy"});
    table2.row(0).format()
            .font_color(tabulate::Color::yellow)
            .font_align(tabulate::FontAlign::center)
            .font_style({tabulate::FontStyle::bold, tabulate::FontStyle::underline});
    std::cout << table2 << std::endl;
```

#  nlohmann-json 

We added function to retrieve variables from json data, and a properly catching and identifying exceptions.

```c++
namespace nlohmann
{
template<typename T>
bool get_value(const nlohmann::json& j, const std::string key, T& value)
{
    try {
        value = j[key];
    } catch (const nlohmann::json::exception& ex) {
        std::cerr << "key = [" << key << "]: " << ex.what() << std::endl;
        return false;
    }
    return true;
}
template<typename T>
bool get_value_v(const nlohmann::json& j,
                 const std::vector<std::string> key_sequence,
                 T& value)
{
    auto k = key_sequence;
    if(key_sequence.empty())
        return false;
    std::string key_str = "key = ['";
    for(uint i = 0; i < k.size() - 1; i++)
        key_str += k.at(i) + "']['";
    key_str += k.back() + "']: ";
    try {
        if(k.size() == 1)
            value = j[k.at(0)];
        if(k.size() == 2)
            value = j[k.at(0)][k.at(1)];
        if(k.size() == 3)
            value = j[k.at(0)][k.at(1)][k.at(2)];
        if(k.size() == 4)
            value = j[k.at(0)][k.at(1)][k.at(2)][k.at(3)];
        if(k.size() == 5)
            value = j[k.at(0)][k.at(1)][k.at(2)][k.at(3)][k.at(4)];
        if(k.size() > 5){
            std::cerr << key_str << "Too long key sequence > 5" << std::endl;
            return false;
        }
    } catch (const nlohmann::json::exception& ex) {
        std::cerr << key_str << ex.what() << std::endl;
        return false;
    }
    return true;
}
}
```

Use case example:

```c++
    nlohmann::json j;
    j["pi"] = 3.141;
    j["happy"] = true;
    j["name"] = "Niels";
    j["nothing"] = nullptr;
    j["answer"]["everything"] = 42;
    j["list"] = { 1, 0, 2 , 7};
    j["object"] = { {"currency", "USD"}, {"value", 42.99} };

    std::string json_data = j.dump(4);
    std::cout << json_data << std::endl;

    int test_int;
    nlohmann::get_value_v(j, {"answer", "everything"}, test_int);
    std::cout << test_int << std::endl;
```


**PS:** template value T for ```get_value()``` and ```get_value_v()``` cannot be an stl container other then std::array.

# plog

Here we extended the ini.h file with functions for easier initialization with fewer code.

```c++

#ifdef EXTEND_ADAT
#include "ColorConsoleAppender.h"
#include "RollingFileAppender.h"

#include "MessageOnlyFormatter.h"
#include "FuncMessageFormatter.h"
#include "CsvFormatter.h"
#include "TxtFormatter.h"


namespace plog
{

class SeverityMessageFormatter
{
public:
    static util::nstring header()
    {
        return util::nstring();
    }

    static util::nstring format(const Record& record)
    {
        util::nostringstream ss;
        ss << PLOG_NSTR("[") << severityToString(record.getSeverity()) << PLOG_NSTR("] ") << record.getMessage() << PLOG_NSTR("\n");

        return ss.str();
    }
};
namespace Init {
enum Formatter{
    allInfo, messageOnly, functionMessage, severityMessage
};


inline bool console_logger(const plog::Severity console_log_severity = Severity::info,
                           const plog::Init::Formatter console_formatter = Formatter::severityMessage)
{
    if(plog::get())
        return false;


    switch (console_formatter) {
    case allInfo:{
        static plog::ColorConsoleAppender<plog::TxtFormatter> consoleAppender;
        plog::init<0>(console_log_severity, &consoleAppender);
        return true;
    }
    case messageOnly:{
        static plog::ColorConsoleAppender<plog::MessageOnlyFormatter> consoleAppender;
        plog::init<0>(console_log_severity, &consoleAppender);
        return true;
    }
    case functionMessage:{
        static plog::ColorConsoleAppender<plog::FuncMessageFormatter> consoleAppender;
        plog::init<0>(console_log_severity, &consoleAppender);
        return true;
    }
    case severityMessage:{
        static plog::ColorConsoleAppender<plog::SeverityMessageFormatter> consoleAppender;
        plog::init<0>(console_log_severity, &consoleAppender);
        return true;
    }
    }
    return false;
}

inline bool console_and_file_logger(const std::string log_file_path,
                                    const plog::Severity console_log_severity = Severity::info,
                                    const plog::Init::Formatter console_formatter = Formatter::severityMessage,
                                    const plog::Severity file_log_severity = Severity::debug){

    //check if logging system is already initiated
    if(plog::get())
        return false;


    switch (console_formatter) {
    case allInfo:{
        static plog::ColorConsoleAppender<plog::TxtFormatter> consoleAppender;
        plog::init<1>(console_log_severity, &consoleAppender);
        break;
    }
    case messageOnly:{
        static plog::ColorConsoleAppender<plog::MessageOnlyFormatter> consoleAppender;
        plog::init<1>(console_log_severity, &consoleAppender);
        break;
    }
    case functionMessage:{
        static plog::ColorConsoleAppender<plog::FuncMessageFormatter> consoleAppender;
        plog::init<1>(console_log_severity, &consoleAppender);
        break;
    }
    case severityMessage:{
        static plog::ColorConsoleAppender<plog::SeverityMessageFormatter> consoleAppender;
        plog::init<1>(console_log_severity, &consoleAppender);
        break;

    }
    }

    const std::string ext = ".csv";
    const size_t max_file_size = 100000;
    const int max_num_files = 10;
    if(log_file_path.compare (log_file_path.length() - ext.length(), ext.length(), ext) == 0){
        static plog::RollingFileAppender<plog::CsvFormatter>
                fileAppender(log_file_path.c_str(), max_file_size, max_num_files);

        plog::init<0>(file_log_severity, &fileAppender).addAppender(plog::get<1>());
        return true;
    }
    static plog::RollingFileAppender<plog::TxtFormatter>
            fileAppender(log_file_path.c_str(), max_file_size, max_num_files);

    plog::init<0>(file_log_severity, &fileAppender).addAppender(plog::get<1>());


    return true;
}

inline void console_logger_if_not_done_yet(
        const plog::Severity console_log_severity = Severity::info,
        const plog::Init::Formatter console_formatter = Formatter::severityMessage)
{
    console_logger(console_log_severity, console_formatter);
}

}  // namespace Init
}  // namespace plog
#endif
```

The following section show how to initialize with one line of code.

```c++
plog::Init::console_and_file_logger("/..path../nice.csv",
                                    plog::Severity::info,
                                    plog::Init::Formatter::severityMessage);


LOGW << "this is a warn";
LOGI << "this is info";
```
