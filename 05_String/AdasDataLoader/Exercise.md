# Exercise

For this exercise, we want to load a data log of a fictional test drive of our autonomous car.  
The collected data is stored in the **data** folder.  
There, the initial data of our ego vehicle is stored in **ego_data.json**, and the data of the other vehicles is stored in **vehicle_data.json**.  
These data logs will always have exactly 1.000 iterations (log entries).

## JSON Library Example

To read in the json data we will use the most used C++ JSON library called **nlohman/json**.  
For this, there is the header-only library file in the **nlohman** subdirectory.

For example, the **ego_data.json** file has the content:

```json
{
    "Lane": 2,
    "Speed": 33.010941520630475
}
```

To load the whole object, use the following code:

```cpp
#include <fstream>

#include "nlohman/json.hpp"

using json = nlohmann::json;

std::ifstream ifs("ego_data.json");
json parsed_data = json::parse(ifs);

float speed_value = static_cast<float>(parsed_data["Speed"]);

std::cout << speed_value << '\n'; // 33.0109415
```

## Code adaptations

Delete the following functions from the previous **AdFunctions.cc** file:

```cpp
void init_ego_vehicle(VehicleType &ego_vehicle);

void init_vehicles(NeighborVehiclesType &vehicles);
```

and implement the following functions in the DataLoader.hpp/.cpp:

```cpp
void init_ego_vehicle(std::string_view filepath, VehicleType &ego_vehicle);

void init_vehicles(std::string_view filepath, NeighborVehiclesType &vehicles);
```

Instead of generating the data by just random numbers, you have to load the JSON data and fill it into the **vehicles** and **ego_vehicle** struct.

- **init_ego_vehicle**
  - Load the starting lane and starting speed from the json file
  - The ID will still be *EGO_VEHICLE_ID* and the distance will still be 0
- **init_vehicles**
  - In these files, there will be the starting lane and a starting distance of the vehicle
  - Only the speed is stored in every cycle (iteration)
  - Load the whole json data into one big static struct

```cpp
static VehiclesData vehicles_data =
    std::vector<std::vector<VehicleType>>(NUM_VEHICLES, std::vector<VehicleType>(NUM_ITERATIONS));
```

- This struct holds for every vehicle (6 in total) and every cycle (1000 in total) one large list of *VehicleType* structs
  - So there are $6 * 1000 = 6000$ *VehicleType* structs
  - Every entry of this struct will then hold as usual the
    - id, lane, speed_mps, distance_m

Afterward, implement the following function:

```cpp
void load_cycle(const std::uint32_t cycle, NeighborVehiclesType &vehicles);
```

- Load the data for the **cycle** from the global static data at the correct index

## Main Function

```cpp
#include <chrono>
#include <cstring>
#include <filesystem>
#include <iostream>
#include <numeric>
#include <thread>

#include "AdFunctions.hpp"
#include "AdTypes.hpp"
#include "DataLoader.hpp"

#include "utils.hpp"

namespace fs = std::filesystem;

int main(int argc, char **argv)
{
    fs::path data_filepath;
    fs::path ego_filepath;

    if (argc != 3)
    {
        data_filepath /= fs::current_path();
        data_filepath /= "data";
        ego_filepath = data_filepath;

        data_filepath /= "vehicle_data.json";
        ego_filepath /= "ego_data.json";
    }
    else
    {
        const auto vehicles_input_path = std::string(argv[1]);
        data_filepath = fs::path(vehicles_input_path);

        const auto ego_input_path = std::string(argv[2]);
        ego_filepath = fs::path(ego_input_path);
    }

    std::uint32_t cycle = 0;
    VehicleType ego_vehicle{};
    NeighborVehiclesType vehicles{};

    init_vehicles(data_filepath.string(), vehicles);
    init_ego_vehicle(ego_filepath.string(), ego_vehicle);

    print_vehicle(ego_vehicle);
    print_neighbor_vehicles(vehicles);

    std::cout << "Start simulation?: ";
    char Input;
    std::cin >> Input;

    while (true)
    {
        clear_console();

        print_scene(ego_vehicle, vehicles);
        compute_future_state(ego_vehicle, vehicles, 0.100F);
        const auto lane_change_request = longitudinal_control(vehicles, ego_vehicle);
        const auto lane_change_successful = lateral_control(vehicles, lane_change_request, ego_vehicle);

        if (lane_change_request != ego_vehicle.lane)
        {
            std::cout << "Lane change request: " << static_cast<std::int32_t>(lane_change_request) << '\n';
        }
        if (lane_change_successful)
        {
            std::cout << "Lane change successull" << '\n';
        }

        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        cycle++;

        load_cycle(cycle, vehicles);
    }

    return 0;
}
```
