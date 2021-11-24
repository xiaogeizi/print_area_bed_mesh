[gcode_macro BED_MESH_CALIBRATE]
# Bed mesh override with automatic probe/bltouch offset calculation
# Works with Klicky Probe on Voron
# November 24, 2021
# Steve Turgeon
rename_existing: _BED_MESH_CALIBRATE
; Do not change any of the existing values below.
variable_parameter_PRINT_MIN : 0,0
variable_parameter_PRINT_MAX : 0,0

variable_parameter_FORCE_NEW_MESH: False ; Do not change

variable_last_area_start_x: -1 ; Do not change
variable_last_area_start_y: -1 ; Do not change
variable_last_area_end_x: -1 ; Do not change
variable_last_area_end_y: -1 ; Do not change

variable_buffer: 20

gcode:

    {% if params.FORCE_NEW_MESH == True %}
        { action_respond_info("Force New Mesh: %s" % (params.FORCE_NEW_MESH)) }
    {% endif %}
    
    {% if printer.toolhead.homed_axes != "xyz" %}
        G28
    {% endif %}
    {% set klicky_available = printer['gcode_macro _Probe_Variables'] != null %}
    {% if params.PRINT_MIN %}
        { action_respond_info("print_min: %s" % params.PRINT_MIN) }
        { action_respond_info("print_max: %s" % params.PRINT_MAX) }
        
        {% set blTouchConfig = printer['configfile'].config["bltouch"] %}
        {% if blTouchConfig %}
            {% set OffsetX = blTouchConfig.x_offset|default(0)|float %}
            {% set OffsetY = blTouchConfig.y_offset|default(0)|float %}
        {% endif %}
        
        {% set probeConfig = printer['configfile'].config["probe"] %}
        {% if probeConfig %}
            {% set OffsetX = probeConfig.x_offset|default(0)|float %}
            {% set OffsetY = probeConfig.y_offset|default(0)|float %}
        {% endif %}

        {% set print_min_x = params.PRINT_MIN.split(",")[0] %}
        {% set print_min_y = params.PRINT_MIN.split(",")[1] %}
        {% set print_max_x = params.PRINT_MAX.split(",")[0] %}
        {% set print_max_y = params.PRINT_MAX.split(",")[1] %}

        {% if last_area_start_x > 0 %}
            { action_respond_info("last_bed_mesh: %s,%s %s,%s" % (last_area_start_x, last_area_start_y, last_area_end_x, last_area_end_y)) }
        {% endif %}

        {% if (params.FORCE_NEW_MESH == True) or (print_min_x|float < last_area_start_x|float) or (print_max_x|float > last_area_end_x|float) or (print_min_y|float < last_area_start_y|float) or (print_max_y|float > last_area_end_y|float)  %}
            {% if klicky_available %}
                _CheckProbe action=query
                Attach_Probe
            {% endif %}
            {% if (print_min_x|default(0)|float < print_max_x|default(0)|float) and (print_min_y|default(0)|float < print_max_y|default(0)|float) %}

                # Get bed_mesh config (probe count, mesh_min and mesh_max for x and y
                {% set bedMeshConfig = printer['configfile'].config["bed_mesh"] %}
                {% set probe_count_x = bedMeshConfig.probe_count.split(",")[0]|int %}
                {% set probe_count_y = bedMeshConfig.probe_count.split(",")[1]|int %}
                {% set relative_reference_index = bedMeshConfig.relative_reference_index %}
                {% set mesh_min_x = bedMeshConfig.mesh_min.split(",")[0]|int %}
                {% set mesh_min_y = bedMeshConfig.mesh_min.split(",")[1]|int %}
                {% set mesh_max_x = bedMeshConfig.mesh_max.split(",")[0]|int %}
                {% set mesh_max_y = bedMeshConfig.mesh_max.split(",")[1]|int %}

                # If print area X is smaller than 50% of the bed size, change to to 3 probe counts for X instead of the default 
                {% if print_max_x|float - print_min_x|float < (mesh_max_x|float - mesh_min_x|float) * 0.50 %}
                    {% set probe_count_x = 3 %}
                {% endif %}

                # If print area Y is smaller than 50% of the bed size, change to to 3 probe counts for Y instead of the default 
                {% if print_max_y|float - print_min_y|float < (mesh_max_y|float - mesh_min_y|float) * 0.50 %}
                    {% set probe_count_y = 3 %}
                {% endif %}

                {% if print_min_x|default(0)|float >=  mesh_min_x %}
                    {% set mesh_min_x = print_min_x|default(0)|float - buffer %}
                {% endif %}

                {% if print_min_y|default(0)|float >=  mesh_min_y %}
                    {% set mesh_min_y = print_min_y|default(0)|float - buffer %}
                {% endif %}

                {% if print_max_x|default(0)|float <=  mesh_max_x %}
                    {% set mesh_max_x = print_max_x|default(0)|float + buffer %}
                {% endif %}

                {% if print_max_y|default(0)|float <=  mesh_max_y %}
                    {% set mesh_max_y = print_max_y|default(0)|float + buffer %}
                {% endif %}

                { action_respond_info("mesh_min: %s,%s" % (mesh_min_x, mesh_min_y)) }
                { action_respond_info("mesh_max: %s,%s" % (mesh_max_x, mesh_max_y)) }
                { action_respond_info("probe_count: %s,%s" % (probe_count_x,probe_count_y)) }

                ; Set variables so they're available outside of macro
                SET_GCODE_VARIABLE MACRO=BED_MESH_CALIBRATE VARIABLE=last_area_start_x VALUE={print_min_x|float}
                SET_GCODE_VARIABLE MACRO=BED_MESH_CALIBRATE VARIABLE=last_area_start_y VALUE={print_min_y|float}
                SET_GCODE_VARIABLE MACRO=BED_MESH_CALIBRATE VARIABLE=last_area_end_x VALUE={print_max_x|float}
                SET_GCODE_VARIABLE MACRO=BED_MESH_CALIBRATE VARIABLE=last_area_end_y VALUE={print_max_y|float}

                {% if (relative_reference_index == 0) %}
                    _BED_MESH_CALIBRATE mesh_min={mesh_min_x|float},{mesh_min_y|float} mesh_max={mesh_max_x|float},{mesh_max_y|float} probe_count={probe_count_x},{probe_count_y}
                {% else %}
                    {% set relative_reference_index = ((probe_count_x * probe_count_y)-1)/2|int %}
                    { action_respond_info("relative_reference_index: %s" % relative_reference_index|int) }
                    _BED_MESH_CALIBRATE mesh_min={mesh_min_x|float},{mesh_min_y|float} mesh_max={mesh_max_x|float},{mesh_max_y|float} probe_count={probe_count_x},{probe_count_y} relative_reference_index={relative_reference_index|int}
                {% endif %}
            {% else %}
                _BED_MESH_CALIBRATE
            {% endif %}
            {% if klicky_available %}
                Dock_Probe
            {% endif %}
        {% else %}
            { action_respond_info("No need to recreate Bed Mesh since it's same as current mesh or smaller") }
        {% endif %}
    {% else %}
        {% if klicky_available %}
            _CheckProbe action=query
            Attach_Probe
        {% endif %}
        _BED_MESH_CALIBRATE
        {% if klicky_available %}
            Dock_Probe
        {% endif %}
    {% endif %}
